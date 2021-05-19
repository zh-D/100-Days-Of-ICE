plugin-request 也很简单，它先导入了 axios 这个包,创建一个 axios 实例：

```javascript
import axios from 'axios';

const axiosInstance = {
  default: axios.create({})
};

// eslint-disable-next-line
function createAxiosInstance(instanceName?: string) {
  return axiosInstance;
}

export default createAxiosInstance;

```

然后有两个方法封装了这个 axios 实例，第一个是 request，它为 class 组件使用；第二个是useRequest，它为 函数组件使用。它们使用的是同一个 axios 实例。

现在看一下 runtime 干了什么。

```javascript
const module = ({ appConfig }) => {
  if (appConfig.request) {
    const { request: requestConfig = {} } = appConfig;
    // 支持配置多实例
    if (Object.prototype.toString.call(requestConfig) === '[object Array]') {
      requestConfig.forEach(requestItem => {
        const instanceName = requestItem.instanceName ? requestItem.instanceName : 'default';
        if (instanceName) {
          const axiosInstance = createAxiosInstance(instanceName)[instanceName];
          setAxiosInstance(requestItem, axiosInstance);
        }
      });
    } else {
      // 配置单一实例
      const axiosInstance = createAxiosInstance().default;
      setAxiosInstance(requestConfig, axiosInstance);
    }
  }
};
```

为什么要配置多实例？

**官方文档说，在某些复杂场景的应用中，我们也可以配置多个请求，每个配置请求都是单一的实例对象。但什么样的复杂场景需要配置多个实例，我不知道。留个坑把。**

然后看一下 setAxiosInstance

```javascript
function setAxiosInstance(requestConfig, axiosInstance) {
  const { interceptors = {}, ...requestOptions } = requestConfig;
  Object.keys(requestOptions).forEach(key => {
    axiosInstance.defaults[key] = requestOptions[key];
  });

  // Add a request interceptor
  if (interceptors.request) {
    axiosInstance.interceptors.request.use(
      interceptors.request.onConfig || function(config){ return config; },
      interceptors.request.onError || function(error) { return Promise.reject(error); }
    );
  }

  // Add a response interceptor
  if (interceptors.response) {
    axiosInstance.interceptors.response.use(
      interceptors.response.onConfig || function(response){ return response; },
      interceptors.response.onError || function(error){ return Promise.reject(error); }
    );
  }
}
```

它这里添加拦截操作：

发送请求前：可以对 RequestConfig 做一些统一处理

请求成功：可以做全局的 toast 展示，或者对 response 做一些格式化

还有很多用法应该看看官方文档。