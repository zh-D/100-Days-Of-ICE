plugin-logger 这个也是很简单，它的 index 从 loglevel 这个包里面拿出这个 logger 对象，导出就完了。

看一下它的 runtime：

```javascript
import * as queryString from 'query-string';
// @ts-ignore
import logger from '$ice/logger';

const module = ({ appConfig }) => {
  const { logger: userLogger = {} } = appConfig;

  if (userLogger.level) {
    return logger.setLevel(userLogger.level);
  }

  let loglevel = process.env.NODE_ENV === 'development' ? 'DEBUG' : 'WARN';
  if (userLogger.smartLoglevel) {
    const searchParams: any = getSearchParams();
    if (searchParams.__loglevel__) {
      loglevel = searchParams.__loglevel__;
    }
  }
  logger.setLevel(loglevel);
};

function getSearchParams() {
  let searchParams = {};
  if (location.hash) {
    searchParams = queryString.parse(location.hash.substring(2));
  } else {
    searchParams = queryString.parse(location.search);
  }
  return searchParams;
}

export default module;
```

它就是区分了开发环境和生产环境，根据环境的不同设置不同的 log 等级。如果是开发环境，就是 debug lever，生产环境就是 warn lever。

setLevel 方法：

A `log.setLevel(level, [persist])` method.

This disables all logging below the given level, so that after a log.setLevel("warn") call log.warn("something") or log.error("something") will output messages, but log.info("something") will not.

