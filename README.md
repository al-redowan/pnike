# CVE-2022-29464
WSO2 RCE (CVE-2022-29464) exploit.
# Details
[CVE-2022-29464](https://docs.wso2.com/display/Security/Security+Advisory+WSO2-2021-1738) is critical vulnerability on WSO2 discovered by [Orange Tsai](https://twitter.com/orange_8361). the vulnerability is an unauthenticated unrestricted arbitrary file upload which which allows unauthenticated attackers to gain RCE on WSO2 servers via uploading malicious JSP files. 

the vulerable upload route is `/fileupload` which is handled by [`FileUploadServlet`](https://github.com/wso2/carbon-kernel/blob/4.4.x/core/org.wso2.carbon.ui/src/main/java/org/wso2/carbon/ui/transports/FileUploadServlet.java)  which is an unprotected route as we can see in the  `indentity.xml` configuration file:
```xml
<Resource context="(.*)/fileupload(.*)" secured="false" http-method="all"/>
```
the `FileUploadServlet` and upon [`init`](https://github.com/wso2/carbon-kernel/blob/d47232dfb2b26c0ef18a74e2ef4aa503caa59697/core/org.wso2.carbon.ui/src/main/java/org/wso2/carbon/ui/transports/FileUploadServlet.java#L71) and through a series of method calls loads eventually from the `carbon.xml` configuration file multiple upload file formats/actions along with the object which hanldes every format.
```java
    public void init(ServletConfig servletConfig) throws ServletException {
        this.servletConfig = servletConfig;
        try {
            fileUploadExecutorManager = new FileUploadExecutorManager(bundleContext, configContext, webContext);
            //Registering FileUploadExecutor Manager as an OSGi service
            bundleContext.registerService(FileUploadExecutorManager.class.getName(), fileUploadExecutorManager, null);
        } catch (CarbonException e) {
            log.error("Exception occurred while trying to initialize FileUploadServlet", e);
            throw new ServletException(e);
        }
    }
```
the [`FileUploadExecutorManager`](https://github.com/wso2/carbon-kernel/blob/d47232dfb2b26c0ef18a74e2ef4aa503caa59697/core/org.wso2.carbon.ui/src/main/java/org/wso2/carbon/ui/transports/fileupload/FileUploadExecutorManager.java#L67) class constructor is as follows:
```java
    public FileUploadExecutorManager(BundleContext bundleContext,
                                     ConfigurationContext configCtx,
                                     String webContext) throws CarbonException {
        this.bundleContext = bundleContext;
        this.configContext = configCtx;
        this.webContext = webContext;
        this.loadExecutorMap();
    }
```
the constructor calls the private method [`loadExecutorMap()`](https://github.com/wso2/carbon-kernel/blob/d47232dfb2b26c0ef18a74e2ef4aa503caa59697/core/org.wso2.carbon.ui/src/main/java/org/wso2/carbon/ui/transports/fileupload/FileUploadExecutorManager.java#L131) which is where the configuration loading is done:
```java
    private void loadExecutorMap() throws CarbonException {
        [snipped]
                try {
            documentElement = XMLUtils.toOM(serverConfiguration.getDocumentElement());
        } catch (Exception e) {
            String msg = "Unable to read Server Configuration.";
            log.error(msg);
            throw new CarbonException(msg, e);
        }
        [snipped]
        OMElement fileUploadConfigElement =
                documentElement.getFirstChildWithName(
                        new QName(ServerConstants.CARBON_SERVER_XML_NAMESPACE, "FileUploadConfig"));
        for (Iterator iterator = fileUploadConfigElement.getChildElements(); iterator.hasNext();) {
            OMElement mapppingElement = (OMElement) iterator.next();
            if (mapppingElement.getLocalName().equalsIgnoreCase("Mapping")) {
                OMElement actionsElement =
                        mapppingElement.getFirstChildWithName(
                                new QName(ServerConstants.CARBON_SERVER_XML_NAMESPACE, "Actions"));
                String confPath = System.getProperty(CarbonBaseConstants.CARBON_CONFIG_DIR_PATH);
        [snipped]
```
the file upload formats configurations is within the `FileUploadConfig` namespace in the XML configuration file, this is the default configuration:
```xml
    <FileUploadConfig>
        <!--
           The total file upload size limit in MB
        -->
        <TotalFileSizeLimit>100</TotalFileSizeLimit>

        <Mapping>
            <Actions>
                <Action>keystore</Action>
                <Action>certificate</Action>
                <Action>*</Action>
            </Actions>
            <Class>org.wso2.carbon.ui.transports.fileupload.AnyFileUploadExecutor</Class>
        </Mapping>

        <Mapping>
            <Actions>
                <Action>jarZip</Action>
            </Actions>
            <Class>org.wso2.carbon.ui.transports.fileupload.JarZipUploadExecutor</Class>
        </Mapping>
        <Mapping>
            <Actions>
                <Action>dbs</Action>
            </Actions>
            <Class>org.wso2.carbon.ui.transports.fileupload.DBSFileUploadExecutor</Class>
        </Mapping>
        <Mapping>
            <Actions>
                <Action>tools</Action>
            </Actions>
            <Class>org.wso2.carbon.ui.transports.fileupload.ToolsFileUploadExecutor</Class>
        </Mapping>
        <Mapping>
            <Actions>
                <Action>toolsAny</Action>
            </Actions>
            <Class>org.wso2.carbon.ui.transports.fileupload.ToolsAnyFileUploadExecutor</Class>
        </Mapping>
    </FileUploadConfig>
```
the `loadExecutorMap()` method creates and fills a `HashMap` of `<Action, Class>` with the Actions and the Classes extracted from the config file. which will be later used to choose which class to use to hanlde properly a given format/action.

Later on when the `/fileupload` route recieves a POST request the [`doPost()`](https://github.com/wso2/carbon-kernel/blob/d47232dfb2b26c0ef18a74e2ef4aa503caa59697/core/org.wso2.carbon.ui/src/main/java/org/wso2/carbon/ui/transports/FileUploadServlet.java#L53) method of the servlet will be called. the method just forwards the request and response object to `execute()` method of `fileUploadExecutorManager` which was intitialized on `init()`
```java
    protected void doPost(HttpServletRequest request,
                          HttpServletResponse response) throws ServletException, IOException {

        try {
            fileUploadExecutorManager.execute(request, response);
        } catch (Exception e) {
            String msg = "File upload failed ";
            log.error(msg, e);
            throw new ServletException(e);
        }
    }
```
the `execute()` method, splits the request url just after the `fileupload/` string, which means it extacts whatever is after the `/fileupload/` in the request URL and it assignes is it to `actionString`.
```java
    public boolean execute(HttpServletRequest request,
                           HttpServletResponse response) throws IOException {

        HttpSession session = request.getSession();
        String cookie = (String) session.getAttribute(ServerConstants.ADMIN_SERVICE_COOKIE);
        request.setAttribute(CarbonConstants.ADMIN_SERVICE_COOKIE, cookie);
        request.setAttribute(CarbonConstants.WEB_CONTEXT, webContext);
        request.setAttribute(CarbonConstants.SERVER_URL,
                             CarbonUIUtil.getServerURL(request.getSession().getServletContext(),
                                                       request.getSession()));


        String requestURI = request.getRequestURI();

        //TODO - fileupload is hardcoded
        int indexToSplit = requestURI.indexOf("fileupload/") + "fileupload/".length();
        String actionString = requestURI.substring(indexToSplit);

        // Register execution handlers
        FileUploadExecutionHandlerManager execHandlerManager =
                new FileUploadExecutionHandlerManager();
        CarbonXmlFileUploadExecHandler carbonXmlExecHandler =
                new CarbonXmlFileUploadExecHandler(request, response, actionString);
        execHandlerManager.addExecHandler(carbonXmlExecHandler);
        OSGiFileUploadExecHandler osgiExecHandler =
                new OSGiFileUploadExecHandler(request, response);
        execHandlerManager.addExecHandler(osgiExecHandler);
        AnyFileUploadExecHandler anyFileExecHandler =
                new AnyFileUploadExecHandler(request, response);
        execHandlerManager.addExecHandler(anyFileExecHandler);
        execHandlerManager.startExec();
        return true;
    }
```
the `actionString` is passed to `CarbonXmlFileUploadExecHandler` class constructor along with `request` and `response`:
```java
        private CarbonXmlFileUploadExecHandler(HttpServletRequest request,
                                               HttpServletResponse response,
                                               String actionString) {
            this.request = request;
            this.response = response;
            this.actionString = actionString;
        }
```
the constructor will save them to its properties. after that `carbonXmlExecHandler` object along with other objects will be added to [`execHandlerManager`](https://github.com/wso2/carbon-kernel/blob/d47232dfb2b26c0ef18a74e2ef4aa503caa59697/core/org.wso2.carbon.ui/src/main/java/org/wso2/carbon/ui/transports/fileupload/FileUploadExecutorManager.java#L298) using `addExecHandler()` method.
```java
        public void addExecHandler(FileUploadExecutionHandler handler) {
            if (prevHandler != null) {
                prevHandler.setNext(handler);
            } else {
                firstHandler = handler;
            }
            prevHandler = handler;
        }
```
then `execHandlerManager.startExec()` is called:
```java
        public void startExec() throws IOException {
            firstHandler.execute();
        }
```
`startExec()` calls [`execute()`](https://github.com/wso2/carbon-kernel/blob/d47232dfb2b26c0ef18a74e2ef4aa503caa59697/core/org.wso2.carbon.ui/src/main/java/org/wso2/carbon/ui/transports/fileupload/FileUploadExecutorManager.java#L430) of the first object added which is `CarbonXmlFileUploadExecHandler`:
```java
        public void execute() throws IOException {
            boolean foundExecutor = false;
            for (String key : executorMap.keySet()) {
                if (key.equals(actionString)) {
                    AbstractFileUploadExecutor obj = executorMap.get(key);
                    foundExecutor = true;
                    obj.executeGeneric(request, response, configContext);
                    break;
                }
            }
            if (!foundExecutor) {
                next();
            }
        }
```
[`execute()`](https://github.com/wso2/carbon-kernel/blob/d47232dfb2b26c0ef18a74e2ef4aa503caa59697/core/org.wso2.carbon.ui/src/main/java/org/wso2/carbon/ui/transports/fileupload/FileUploadExecutorManager.java#L430) loops trough the `HashMap` of `<Action, Class>` created earlier and finds the Action (key) which is equalt to `actionString`, if found the `executeGeneric()` method of the object associated with that Action will be called.
to revise the default configuration has 7 actions which are:
* `keystore`, `certificate`, `*` handles by `org.wso2.carbon.ui.transports.fileupload.AnyFileUploadExecutor`
* `jarZip` handled by `org.wso2.carbon.ui.transports.fileupload.JarZipUploadExecutor`
* `dbs` handled by `org.wso2.carbon.ui.transports.fileupload.DBSFileUploadExecutor`
* `tools` handled by `org.wso2.carbon.ui.transports.fileupload.ToolsFileUploadExecutor`
* `toolsAny` handled by `org.wso2.carbon.ui.transports.fileupload.ToolsAnyFileUploadExecutor`
each of this objects does handles the upload differently some of them accepts specific extensions. 

the first one i found vulnerable to arbitraty file write was `toolsAny` ([`ToolsAnyFileUploadExecutor`](https://github.com/wso2/carbon-kernel/blob/4.4.x/core/org.wso2.carbon.ui/src/main/java/org/wso2/carbon/ui/transports/fileupload/ToolsAnyFileUploadExecutor.java)).
`ToolsAnyFileUploadExecutor` does not have a `executeGeneric()` method but it extends [`AbstractFileUploadExecutor`](https://github.com/wso2/carbon-kernel/blob/d47232dfb2b26c0ef18a74e2ef4aa503caa59697/core/org.wso2.carbon.ui/src/main/java/org/wso2/carbon/ui/transports/fileupload/AbstractFileUploadExecutor.java#L61) which does have a [`executeGeneric()`](https://github.com/wso2/carbon-kernel/blob/d47232dfb2b26c0ef18a74e2ef4aa503caa59697/core/org.wso2.carbon.ui/src/main/java/org/wso2/carbon/ui/transports/fileupload/AbstractFileUploadExecutor.java#L97) method:
```java
    boolean executeGeneric(HttpServletRequest request,
                           HttpServletResponse response,
                           ConfigurationContext configurationContext) throws IOException {//,
        //    CarbonException {
        this.configurationContext = configurationContext;
        try {
            parseRequest(request);
            return execute(request, response);
        } catch (FileUploadFailedException e) {
            sendErrorRedirect(request, response, e);
        } catch (FileSizeLimitExceededException e) {
            sendErrorRedirect(request, response, e);
        } catch (CarbonException e) {
            sendErrorRedirect(request, response, e);
        }
        return false;
    }
```
`executeGeneric()` calls first `parseRequest()` with the request object as a parameter:
```java
    protected void parseRequest(HttpServletRequest request) throws FileUploadFailedException,
                                                                 FileSizeLimitExceededException {
        fileItemsMap.set(new HashMap<String, ArrayList<FileItemData>>());
        formFieldsMap.set(new HashMap<String, ArrayList<String>>());

        ServletRequestContext servletRequestContext = new ServletRequestContext(request);
        boolean isMultipart = ServletFileUpload.isMultipartContent(servletRequestContext);
        Long totalFileSize = 0L;

        if (isMultipart) {

            List items;
            try {
                items = parseRequest(servletRequestContext);
            } catch (FileUploadException e) {
                String msg = "File upload failed";
                log.error(msg, e);
                throw new FileUploadFailedException(msg, e);
            }
            boolean multiItems = false;
            if (items.size() > 1) {
                multiItems = true;
            }

            // Add the uploaded items to the corresponding maps.
            for (Iterator iter = items.iterator(); iter.hasNext();) {
                FileItem item = (FileItem) iter.next();
                String fieldName = item.getFieldName().trim();
                if (item.isFormField()) {
                    if (formFieldsMap.get().get(fieldName) == null) {
                        formFieldsMap.get().put(fieldName, new ArrayList<String>());
                    }
                    try {
                        formFieldsMap.get().get(fieldName).add(new String(item.get(), "UTF-8"));
                    } catch (UnsupportedEncodingException ignore) {
                    }
                } else {
                    String fileName = item.getName();
                    if ((fileName == null || fileName.length() == 0) && multiItems) {
                        continue;
                    }
                    if (fileItemsMap.get().get(fieldName) == null) {
                        fileItemsMap.get().put(fieldName, new ArrayList<FileItemData>());
                    }
                    totalFileSize += item.getSize();
                    if (totalFileSize < totalFileUploadSizeLimit) {
                        fileItemsMap.get().get(fieldName).add(new FileItemData(item));
                    } else {
                        throw new FileSizeLimitExceededException(getFileSizeLimit() / 1024 / 1024);
                    }
                }
            }
        }
    }
```
it first assures that the POST request is a multipart POST request, and then extarcts the uploaded files, assures that the POST request contains at least on uploaded file and validates it against the maximum file size.
after returning from `parseRequest()`, `executeGeneric()` will call now the `execute()` method which is [overrode](https://github.com/wso2/carbon-kernel/blob/d47232dfb2b26c0ef18a74e2ef4aa503caa59697/core/org.wso2.carbon.ui/src/main/java/org/wso2/carbon/ui/transports/fileupload/ToolsAnyFileUploadExecutor.java#L36) by `ToolsAnyFileUploadExecutor`:
```java
	@Override
	public boolean execute(HttpServletRequest request,
			HttpServletResponse response) throws CarbonException, IOException {
		PrintWriter out = response.getWriter();
        try {
        	Map fileResourceMap =
                (Map) configurationContext
                        .getProperty(ServerConstants.FILE_RESOURCE_MAP);
        	if (fileResourceMap == null) {
        		fileResourceMap = new TreeBidiMap();
        		configurationContext.setProperty(ServerConstants.FILE_RESOURCE_MAP,
                                             fileResourceMap);
        	}
            List<FileItemData> fileItems = getAllFileItems();
            //String filePaths = "";

            for (FileItemData fileItem : fileItems) {
                String uuid = String.valueOf(
                        System.currentTimeMillis() + Math.random());
                String serviceUploadDir =
                        configurationContext
                                .getProperty(ServerConstants.WORK_DIR) +
                                File.separator +
                                "extra" + File
                                .separator +
                                uuid + File.separator;
                File dir = new File(serviceUploadDir);
                if (!dir.exists()) {
                    dir.mkdirs();
                }
                File uploadedFile = new File(dir, fileItem.getFileItem().getFieldName());
                try (FileOutputStream fileOutStream = new FileOutputStream(uploadedFile)) {
                    fileItem.getDataHandler().writeTo(fileOutStream);
                    fileOutStream.flush();
                }
                response.setContentType("text/plain; charset=utf-8");
                //filePaths = filePaths + uploadedFile.getAbsolutePath() + ",";
                fileResourceMap.put(uuid, uploadedFile.getAbsolutePath());
                out.write(uuid);
            }
            //filePaths = filePaths.substring(0, filePaths.length() - 1);
            //out.write(filePaths);
            out.flush();
        } catch (Exception e) {
            log.error("File upload FAILED", e);
            out.write("<script type=\"text/javascript\">" +
                    "top.wso2.wsf.Util.alertWarning('File upload FAILED. File may be non-existent or invalid.');" +
                    "</script>");
        } finally {
            out.close();
        }
        return true;
	}
```
Here is where the bug lies, `execute()` method is vulnerable to a path traversal vulenerabulity as it trusts the filename given by the user in the POST request. without the path traversal escaping the tmp dir the file is actually saved to:
```
./tmp/work/extra/$uuid/$filename
```
with `uuid` being returned in the response:

![image](https://user-images.githubusercontent.com/67718634/164338439-5cc7871c-9bab-4c7d-ba98-d243e8d8ad39.png)

the file can be found in:

![image](https://user-images.githubusercontent.com/67718634/164338596-8d999333-6d8b-4d6d-a082-934c681033fa.png)

Now we just need to escape the `tmp` directory and add our JSP shell to some location being served by the WSO2. one of these location is `./repository/deployment/server/webapps`:
![image](https://user-images.githubusercontent.com/67718634/164339558-1d300cf5-d113-4702-bd46-649abd8e666f.png)

this directory contains multiple deployed WAR applications and also raw WAR files. one of those WAR applications is the `authenticationendpoint` which handles the authentication to WSO2 and its location is `./repository/deployment/server/webapps/authenticationendpoint`:
![image](https://user-images.githubusercontent.com/67718634/164339712-3a69dc16-7164-425c-8cdb-c978c883f4cb.png)

# PoC
![image](https://user-images.githubusercontent.com/67718634/164339943-588e6879-e93c-4392-9300-d51c6ccb3fe9.png)

![image](https://user-images.githubusercontent.com/67718634/164340091-70bfd623-526d-45a5-a9ee-89912a4a1cd2.png)

![image](https://user-images.githubusercontent.com/67718634/164340406-8c18a647-5e67-4a21-b426-964b8c0c63f6.png)
