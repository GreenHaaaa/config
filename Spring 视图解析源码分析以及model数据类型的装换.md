# 视图的解析
首先看DisPatchServlet的Render（）方法

```java
    protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
        Locale locale = this.localeResolver != null ? this.localeResolver.resolveLocale(request) : request.getLocale();
        response.setLocale(locale);
        String viewName = mv.getViewName();
        View view;
        if (viewName != null) {
            view = this.resolveViewName(viewName, mv.getModelInternal(), locale, request);
            if (view == null) {
                throw new ServletException("Could not resolve view with name '" + mv.getViewName() + "' in servlet with name '" + this.getServletName() + "'");
            }
        } else {
            view = mv.getView();
            if (view == null) {
                throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a View object in servlet with name '" + this.getServletName() + "'");
            }
        }

        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Rendering view [" + view + "] in DispatcherServlet with name '" + this.getServletName() + "'");
        }

        try {
            if (mv.getStatus() != null) {
                response.setStatus(mv.getStatus().value());
            }

            view.render(mv.getModelInternal(), request, response);
        } catch (Exception var8) {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Error rendering view [" + view + "] in DispatcherServlet with name '" + this.getServletName() + "'", var8);
            }

            throw var8;
        }
    }
```
1. 获取视图
```
 view = this.resolveViewName(viewName, mv.getModelInternal(), locale, request);
```
2.对视图数据进行装换

```
view.render(mv.getModelInternal(), request, response);
```

## 视图获取
首先，根据项目的配置，会有多个ViewResolover进行视图的解析，下面的分析以ContentNegotiatingViewResolver为例
在DispatchServlet的resolveViewName中，会遍历上述的ViewResolver，在ContentNegotiatingViewResolver这个视图解析器中的resolveViewName的方法中，代码如下：

```java
public View resolveViewName(String viewName, Locale locale) throws Exception {
        RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
        Assert.state(attrs instanceof ServletRequestAttributes, "No current ServletRequestAttributes");
        List<MediaType> requestedMediaTypes = this.getMediaTypes(((ServletRequestAttributes)attrs).getRequest());
        if (requestedMediaTypes != null) {
            List<View> candidateViews = this.getCandidateViews(viewName, locale, requestedMediaTypes);
            View bestView = this.getBestView(candidateViews, requestedMediaTypes, attrs);
            if (bestView != null) {
                return bestView;
            }
        }

        if (this.useNotAcceptableStatusCode) {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("No acceptable view found; returning 406 (Not Acceptable) status code");
            }

            return NOT_ACCEPTABLE_VIEW;
        } else {
            this.logger.debug("No acceptable view found; returning null");
            return null;
        }
    }
```
1. 获取到request中的accepted中的视图类型以及produce的视图类型

```
 protected List<MediaType> getMediaTypes(HttpServletRequest request) {
        Assert.state(this.contentNegotiationManager != null, "No ContentNegotiationManager set");

        try {
            ServletWebRequest webRequest = new ServletWebRequest(request);
            List<MediaType> acceptableMediaTypes = this.contentNegotiationManager.resolveMediaTypes(webRequest);
            List<MediaType> producibleMediaTypes = this.getProducibleMediaTypes(request);
            Set<MediaType> compatibleMediaTypes = new LinkedHashSet();
            Iterator var6 = acceptableMediaTypes.iterator();

            while(var6.hasNext()) {
                MediaType acceptable = (MediaType)var6.next();
                Iterator var8 = producibleMediaTypes.iterator();

                while(var8.hasNext()) {
                    MediaType producible = (MediaType)var8.next();
                    if (acceptable.isCompatibleWith(producible)) {
                        compatibleMediaTypes.add(this.getMostSpecificMediaType(acceptable, producible));
                    }
                }
            }

            List<MediaType> selectedMediaTypes = new ArrayList(compatibleMediaTypes);
            MediaType.sortBySpecificityAndQuality(selectedMediaTypes);
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Requested media types are " + selectedMediaTypes + " based on Accept header types and producible media types " + producibleMediaTypes + ")");
            }

            return selectedMediaTypes;
        } catch (HttpMediaTypeNotAcceptableException var10) {
            return null;
        }
    }
```

2. getCandidateViews方法根据XML中预设的viewResolvers匹配1中的MediaType获取候选View，同时增加XML中defaultViews中的view。每个view都有对应的content-type，大部分的视图继承于AbstractView，默认值都是text/html;charset=ISO-8859-1。有些特定视图，比如MappingJackson2JsonView会重设contentType为application/json。每个view中的contentType在最后匹配中都会实际用到，所以很重要。

在此方法中，会调用每一个viewResolvers的resolveViewName方法创建view，需要传入的参数为viewname与locale，创建过程中，

会先从cache中查看是否已经匹配过，若存在，直接返回view；若不存在，继续创建view。

第一步：检测viewResolvers中的viewNames属性，<property name="viewNames" value=".ftl"/>（支持正则表达式 如*.ftl,），

表达式：return (viewNames == null || PatternMatchUtils.simpleMatch(viewNames, viewName));

若未设置viewNames属性，则默认为null，允许创建，进入第二步；若设置，会根据viewNames进行正则表达式的匹配，若匹配继续第二步，若不匹配，返回null view，跳出此viewResolver；继续与下一个viewresolvers进行匹配并获得view。

第二步：进行每个viewResolve都有的view的创建过程（初始化content-type,记录最终跳转页面prefix，postfix等），但诸如Freemarker，velocity 的 viewResolvers都会在view创建过程中的最后一步调用重写checkResource方法，查看最终跳转的ftl或vm文件是否存在？若不存在，则将创建的视图View重设成null并返回，故无法匹配并加入候选视图。

而InternalResourceView的checkResource没有重写，调用父类AbstractUrlBasedView的checkResource，默认返回true，为所有请求创建view，无论是否有最终文件。所以，InternalResourceView一般都作为springmvc最后的视图解析器，来处理一切请求。
```java
private List<View> getCandidateViews(String viewName, Locale locale, List<MediaType> requestedMediaTypes) throws Exception {
        List<View> candidateViews = new ArrayList();
        if (this.viewResolvers != null) {
            Assert.state(this.contentNegotiationManager != null, "No ContentNegotiationManager set");
            Iterator var5 = this.viewResolvers.iterator();

            while(var5.hasNext()) {
                ViewResolver viewResolver = (ViewResolver)var5.next();
                View view = viewResolver.resolveViewName(viewName, locale);
                if (view != null) {
                    candidateViews.add(view);
                }

                Iterator var8 = requestedMediaTypes.iterator();

                while(var8.hasNext()) {
                    MediaType requestedMediaType = (MediaType)var8.next();
                    List<String> extensions = this.contentNegotiationManager.resolveFileExtensions(requestedMediaType);
                    Iterator var11 = extensions.iterator();

                    while(var11.hasNext()) {
                        String extension = (String)var11.next();
                        String viewNameWithExtension = viewName + '.' + extension;
                        view = viewResolver.resolveViewName(viewNameWithExtension, locale);
                        if (view != null) {
                            candidateViews.add(view);
                        }
                    }
                }
            }
        }

        if (!CollectionUtils.isEmpty(this.defaultViews)) {
            candidateViews.addAll(this.defaultViews);
        }

        return candidateViews;
    }
```
3. getBestView方法获取最符合条件的匹配视图

首先，如果是smartView，即redirectView则return

否则：筛选getMediaTypes返回的content-type类型，与candidateViews中的候选视图匹配，其中：

下面的contentType重写了MappingJackson2JsonView的contentType，用于匹配cnManager中的mediaTypes，若该视图可以解析此contenttype，则返回

例子：若访问localhost/xxx/a.json，匹配策略为后缀匹配（favorExtension开启），由于后缀为json，则匹配cnManager中的text/plain筛选view后发现匹配视图为MappingJackson2JsonView，返回json格式的数据；若访问localhost/xxx/a,由于无后缀及参数匹配，则匹配策略为accept-header匹配，由于content-type为text/html，则匹配InternalResourceView中的视图，返回jsp页面。

你可能注意到，上述viewResolvers中配置了两个视图，为什么匹配InternalResourceView,而不是freeMarker？这是因为进入候选试图的只有InternalResourceView视图解析器，具体原因参见getCandidateViews。

```java
private View getBestView(List<View> candidateViews, List<MediaType> requestedMediaTypes, RequestAttributes attrs) {
        Iterator var4 = candidateViews.iterator();

        while(var4.hasNext()) {
            View candidateView = (View)var4.next();
            if (candidateView instanceof SmartView) {
                SmartView smartView = (SmartView)candidateView;
                if (smartView.isRedirectView()) {
                    if (this.logger.isDebugEnabled()) {
                        this.logger.debug("Returning redirect view [" + candidateView + "]");
                    }

                    return candidateView;
                }
            }
        }

        var4 = requestedMediaTypes.iterator();

        while(var4.hasNext()) {
            MediaType mediaType = (MediaType)var4.next();
            Iterator var10 = candidateViews.iterator();

            while(var10.hasNext()) {
                View candidateView = (View)var10.next();
                if (StringUtils.hasText(candidateView.getContentType())) {
                    MediaType candidateContentType = MediaType.parseMediaType(candidateView.getContentType());
                    if (mediaType.isCompatibleWith(candidateContentType)) {
                        if (this.logger.isDebugEnabled()) {
                            this.logger.debug("Returning [" + candidateView + "] based on requested media type '" + mediaType + "'");
                        }

                        attrs.setAttribute(View.SELECTED_CONTENT_TYPE, mediaType, 0);
                        return candidateView;
                    }
                }
            }
        }

        return null;
    }
```
## 数据的装换
数据类型装换这一步是在AbstractView的render方法中进行，主要分为三个方法

```java
 public void render(@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Rendering view with name '" + this.beanName + "' with model " + model + " and static attributes " + this.staticAttributes);
        }

        Map<String, Object> mergedModel = this.createMergedOutputModel(model, request, response);
        this.prepareResponse(request, response);
        this.renderMergedOutputModel(mergedModel, this.getRequestToExpose(request), response);
    }
```
1. 这个方法将model，request中的信息合并到一个model（LinkHashMap）中

```java
protected Map<String, Object> createMergedOutputModel(@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) {
        Map<String, Object> pathVars = this.exposePathVariables ? (Map)request.getAttribute(View.PATH_VARIABLES) : null;
        int size = this.staticAttributes.size();
        size += model != null ? model.size() : 0;
        size += pathVars != null ? pathVars.size() : 0;
        Map<String, Object> mergedModel = new LinkedHashMap(size);
        mergedModel.putAll(this.staticAttributes);
        if (pathVars != null) {
            mergedModel.putAll(pathVars);
        }

        if (model != null) {
            mergedModel.putAll(model);
        }

        if (this.requestContextAttribute != null) {
            mergedModel.put(this.requestContextAttribute, this.createRequestContext(request, response, mergedModel));
        }

        return mergedModel;
    }
```
2. 这个方法为response设置content-type以及CharacterCode。

```java
 protected void prepareResponse(HttpServletRequest request, HttpServletResponse response) {
        if (this.generatesDownloadContent()) {
            response.setHeader("Pragma", "private");
            response.setHeader("Cache-Control", "private, must-revalidate");
        }

    }
```
3. 这个方法是将model中数据进行格式转换的主要方法，在AbstractView中的是一个抽象方法，由转换数据格式的不同而调用不同的实现类

```java
protected void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
        ByteArrayOutputStream temporaryStream = null;
        Object stream;
        if (this.updateContentLength) {
            temporaryStream = this.createTemporaryOutputStream();
            stream = temporaryStream;
        } else {
            stream = response.getOutputStream();
        }

        Object value = this.filterAndWrapModel(model, request);
        this.writeContent((OutputStream)stream, value);
        if (temporaryStream != null) {
            this.writeToResponse(response, temporaryStream);
        }

    }
```
