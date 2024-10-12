页面的性能情况：包括各阶段加载耗时，一些关键性的用户体验指标等
用户的行为情况：包括PV、UV、访问来路，路由跳转等
接口的调用情况：通过http访问的外部接口的成功率、耗时情况等
页面的稳定情况：各种前端异常等
数据上报及优化：如何将监控捕获到的数据优雅的上报

如果全部内容都放在一篇文章中进行说明，会导致过于冗长且失去层级，所以本篇中只介绍 前端监控的第一部分：页面性能监控；
为什么要搞自建监控
前端监控是一个历史长久的话题了，目前一提到监控平台，大部分开发同学想到的是什么？没错，sentry，那么我们这篇文章为什么要自己搞呢？

方便团队做自定义的UV用户识别，比如通过登录账号ID或者通过设备信息；甚至从设备信息转入登录态后的继承
方便接入自己团队的各种告警业务等
方便做各维度数据的联合分析，比如发生错误可以联动查询用户行为追溯数据等
方便做业务需求上的拓展，比如自定义埋点、特殊的数据分析维度
方便前后端全链路的一个API请求链路分析；

系列文章传送门
一文摸清前端监控实践要点（一）性能监控
一文摸清前端监控实践要点（二）行为监控
一文摸清前端监控实践要点（三）错误监控
腾讯三面：说说前端监控告警分析平台的架构设计和难点亮点？
页面的性能情况
我们都听说过性能的重要性。但当我们谈起性能，以及让网站"速度提升"时，我们具体指的是什么？
其实性能是相对的：

某个网站可能对一个用户来说速度很快（网速快，设备强大的情况下），但可能对另一个用户来说速度很慢（网速慢，设备低端的情况下）。
两个网站完成加载所需的时间或许相同，但其中一个却 显得 加载速度更快（如果该网站逐步加载内容，而不是等到最后才一起显示）。
一个网站可能 看起来 加载速度很快，但随后对用户交互的响应速度却很慢（或根本无响应）。

因此，在谈论性能时，重要的是做到精确，并且根据能够进行定量测量的客观标准来论及性能。这些标准就是 指标。
而 前端性能监控，就是要监测页面的性能情况，将各种的性能数据指标量化并收集
W3C标准化
官方地址：Navigation Timing Level 2
为了帮助开发者更好地衡量和改进前端页面性能，W3C性能小组引入了 Navigation Timing API ，实现了自动、精准的页面性能打点；开发者可以通过 window.performance 属性获取。
下图是W3C第一版的 Navigation Timing 的处理模型。

上图 Level 1 的规范，2012 年底进入候选建议阶段，至今仍在日常使用中；但是在W3C的议程上，它已经功成身退，让位给了精度更高，功能更强大，层次更分明的 Level 2（处理模型如下图）。比如独立划分出来的 Resource Timing，使得我们可以获取具体资源的详细耗时信息。
w3c level2 扩充了 performance 的定义，并增加了 PerformanceObserver 的支持。

图中指标的解读可以在 developer.mozilla.org/zh-CN/docs/… 中查看
web-vitals
web-vitals 是一个 google 开源的 一个用以衡量性能和用户体验的工具，我将它放在这里介绍，是因为下文的性能关键指标获取中，有的会介绍通过这个开源插件进行获取的方法；
相比于我们自己手动写，它会替我们覆盖很多兼容和特殊的场景；

它支持了这些指标：web.dev/metrics/

整体封装
有的同学跟我说想看一下整体的封装和初始化应该怎么写，那么这里就在写具体的每个性能指标获取之前，先对数据的暂存以及每个函数的初始化位置做一下处理：
然后下文的每个指标讲解中，最后都会附上封装的参考：
数据暂存
js 代码解读复制代码// store.ts
export enum metricsName {
  FP = 'first-paint',
  FCP = 'first-contentful-paint',
  LCP = 'largest-contentful-paint',
  FID = 'first-input-delay',
  CLS = 'cumulative-layout-shift',
  NT = 'navigation-timing',
  RF = 'resource-flow',
}

export interface IMetrics {
  [prop: string | number]: any;
}

// Map 暂存数据
export default class metricsStore {
  state: Map<metricsName | string, IMetrics>;

  constructor() {
    this.state = new Map<metricsName | string, IMetrics>();
  }

  set(key: metricsName | string, value: IMetrics): void {
    this.state.set(key, value);
  }

  add(key: metricsName | string, value: IMetrics): void {
    const keyValue = this.state.get(key);
    this.state.set(key, keyValue ? keyValue.concat([value]) : [value]);
  }

  get(key: metricsName | string): IMetrics | undefined {
    return this.state.get(key);
  }

  has(key: metricsName | string): boolean {
    return this.state.has(key);
  }

  clear() {
    this.state.clear();
  }

  getValues(): IMetrics {
    // Map 转为 对象 返回
    return Object.fromEntries(this.state);
  }
}


整体初始化
js 代码解读复制代码import MetricsStore, { metricsName, IMetrics } from './store';

export interface PerformanceEntryHandler {
  (entry: any): void;
}

export const afterLoad = (callback: any) => {
  if (document.readyState === 'complete') {
    setTimeout(callback);
  } else {
    window.addEventListener('pageshow', callback, { once: true, capture: true });
  }
};

export const observe = (type: string, callback: PerformanceEntryHandler): PerformanceObserver | undefined => {
  // 类型合规，就返回 observe
  if (PerformanceObserver.supportedEntryTypes?.includes(type)) {
    const ob: PerformanceObserver = new PerformanceObserver((l) => l.getEntries().map(callback));

    ob.observe({ type, buffered: true });
    return ob;
  }
  return undefined;
};

// 初始化入口，外部调用只需要 new WebVitals();
export default class WebVitals {
  private engineInstance: EngineInstance;

  // 本地暂存数据在 Map 里 （也可以自己用对象来存储）
  public metrics: MetricsStore;

  constructor(engineInstance: EngineInstance) {
    this.engineInstance = engineInstance;
    this.metrics = new MetricsStore();
    this.initLCP();
    this.initCLS();
    this.initResourceFlow();

    // 这里的 FP/FCP/FID需要在页面成功加载了再进行获取
    afterLoad(() => {
      this.initNavigationTiming();
      this.initFP();
      this.initFCP();
      this.initFID();
      this.perfSendHandler();
    });
  }
  
  // 性能数据的上报策略
  perfSendHandler = (): void => {
    // 如果你要监听 FID 数据。你就需要等待 FID 参数捕获完成后进行上报;
    // 如果不需要监听 FID，那么这里你就可以发起上报请求了;
  };

  // 初始化 FP 的获取以及返回
  initFP = (): void => {
    //... 详情代码在下文
  };

  // 初始化 FCP 的获取以及返回
  initFCP = (): void => {
    //... 详情代码在下文
  };

  // 初始化 LCP 的获取以及返回
  initLCP = (): void => {
    //... 详情代码在下文
  };

  // 初始化 FID 的获取 及返回
  initFID = (): void => {
    //... 详情代码在下文
  };

  // 初始化 CLS 的获取以及返回
  initCLS = (): void => {
    //... 详情代码在下文
  };

  // 初始化 NT 的获取以及返回
  initNavigationTiming = (): void => {
    //... 详情代码在下文
  };

  // 初始化 RF 的获取以及返回
  initResourceFlow = (): void => {
    //... 详情代码在下文
  };
}

获取 PerformanceTiming
既然已经有了 W3C 标准化定义的 PerformanceTiming ，那我们自然就要获取它并予以活用，用来计算获取我们所需要的性能指标
而对于上文提到的 W3C Level1 和 W3C Level 2；建议先使用 W3C Performance Timeline Level 2 的 High-Resolution Time，时间精度可以达毫秒的小数点好几位，当浏览器不支持时获取结果为空数组。所以还得向下兼容使用 W3C Level1;
js 代码解读复制代码let timing =
    // W3C Level2  PerformanceNavigationTiming
    // 使用了High-Resolution Time，时间精度可以达毫秒的小数点好几位。
    performance.getEntriesByType('navigation').length > 0
      ? performance.getEntriesByType('navigation')[0]
      : performance.timing; // W3C Level1  (目前兼容性高，仍然可使用，未来可能被废弃)。

以用户为中心的性能指标
什么叫以用户为中心的性能指标呢？其实就是可以直接的体现出用户的使用体验的指标；目前 Google 定义了FCP、LCP、CLS 等体验指标，已经成为了目前业界的标准；对于用户体验来说，指标可以简单归纳为 加载速度、视觉稳定、交互延迟等几个方面；

加载速度 决定了 用户是否可以尽早感受到页面已经加载完成
视觉稳定 衡量了 页面上的视觉变化对用户造成的负面影响大小
交互延迟 决定了 用户是否可以尽早感受到页面已经可以操作

而针对于上面三个方面，我们可以采集以下 多种指标 进行衡量
白屏（FP）、灰屏（FCP）
W3C标准化在 w3c/paint-timing 定义了 首次非网页背景像素渲染（fp）(白屏时间)，我们可以直接去取;
js 代码解读复制代码new PerformanceObserver((entryList) => {
  for (const entry of entryList.getEntriesByName('first-paint')) {
    console.log('fp', entry);
  }
}).observe({ type: 'paint', buffered: true });

paint 还定义了一个 首次内容渲染（fcp)(灰屏时间)，这里简单说一下获取方式:

我们可以自己手动写一个 FCP 获取：

js 代码解读复制代码new PerformanceObserver((entryList) => {
  for (const entry of entryList.getEntriesByName('first-contentful-paint')) {
    console.log('fcp', entry);
  }
}).observe({ type: 'paint', buffered: true });


也可以使用 google 的 web-vitals ：

js 代码解读复制代码import {getFCP} from 'web-vitals';
// 当 FCP 可用时立即进行测量和记录。
getFCP(console.log);


还可以 简单封装一下 以适合上文的整体封装

ts 代码解读复制代码// 获取 FP
export const getFP = (): PerformanceEntry | undefined => {
  const [entry] = performance.getEntriesByName('first-paint');
  return entry;
};

// 初始化 FP 的获取以及返回
initFP = (): void => {
  const entry = getFP();
  const metrics = {
    startTime: entry?.startTime.toFixed(2),
    entry,
  } as IMetrics;
  this.metrics.set(metricsName.FP, metrics);
};

// 获取 FCP
export const getFCP = (): PerformanceEntry | undefined => {
  const [entry] = performance.getEntriesByName('first-contentful-paint');
  return entry;
};

// 初始化 FCP 的获取以及返回
initFCP = (): void => {
  const entry = getFCP();
  const metrics = {
    startTime: entry?.startTime.toFixed(2),
    entry,
  } as IMetrics;
  this.metrics.set(metricsName.FCP, metrics);
};


FP首次绘制：页面视觉首次发生变化的时间点。比如设置的body背景色；FP不包含默认背景绘制，但包含非默认的背景绘制。
FCP首次内容绘制：首次绘制任何文本、图像、非空白canvas或者SVG的时间点。
FP和FCP的区别：FCP是首次绘制有效内容的时间点；所以FP会等于或者先于FCP
但假如，我给单页面应用的 body 元素加了一个背景色，那么FP记录的时间就是开始绘制带背景色的body的时间点，而FCP记录的则是 body 生成之后，首次绘制来自DOM的有效内容的时间点，这个时候FP的时间点就先于FCP

首次有效绘制（FMP）(首屏）
有的同学或许会疑问，上面的灰屏时间，在我看来就是首屏的时间呀？为什么还需要单独开一栏说呢？其实不然，我们再回忆一次灰屏的定义：FCP是首次绘制任何文本、图像、非空白canvas或者SVG的时间点。注意，这里的用词是 任何 ；也就意味着，如果按照这种逻辑，客户端渲染了一个字的时间点，就认为了首屏时间的时间点所在，
FMP 首次有效绘制，我们可以定义为 页面渲染过中 元素增量最大的点，因为元素增量最大的时候，页面主要内容也就一般都渲染完成了；
目前 W3C 还没有关于 FMP 的标准化计算定义；但是W3C关于首屏统计已经进入了提议阶段，坐等W3C再次标准化。可以在github上看到最新的进展, 详见 w3c/paint-timing。
如果想系统化首屏的计算，可以参考 阿里ARMS的FMP文章 ，或者 可以使用手动在代码中埋点的方式进行计算；
至于如果你想用 MutationObserver 进行自己写一个算法，可以按照我下面的思路进行自己实现
js 代码解读复制代码export const initFMP = (): void => {
  new MutationObserver((records: Array<MutationRecord>) => {
    // 对当前的 document 进行计算评分
    // 或者对 records.addedNodes的每个 node 元素，计算评分累加;每次遍历元素还需要判断此元素是否在可视区域
  }).observe(document, { childList: true, subtree: true });
};

最大内容绘制（LCP）
LCP 是页面内首次开始加载的时间点，到 可视区域内最大的图像或者文本块完成渲染 的 相对时间，是一个以用户为中心的性能指标，可以测试用户主观感知到的页面加载速度，因为最大内容绘制完成时，往往可以认为 页面将要加载完成

通常来说，为了提供良好的用户体验，我们应该努力将 最大内容绘制控（LCP） 制在2.5 秒或以内。

我们可以自己手动写一个LCP获取：

js 代码解读复制代码new PerformanceObserver((entryList) => {
  const entries = entryList.getEntries();
  const entry = entries[entries.length - 1];
  console.log('lcp', entry);
}).observe({ type: 'largest-contentful-paint', buffered: true });


也可以使用 google 的 web-vitals ：

js 代码解读复制代码import { getLCP } from 'web-vitals';
// 当 LCP 可用时立即进行测量和记录。
getLCP(console.log);


还可以 简单封装一下 以适合上文的整体封装

js 代码解读复制代码// 获取 LCP
export const getLCP = (entryHandler: PerformanceEntryHandler): PerformanceObserver | undefined => {
  return observe('largest-contentful-paint', entryHandler);
};

// 初始化 LCP 的获取以及返回
initLCP = (): void => {
  const entryHandler = (entry: PerformanceEntry) => {
    const metrics = {
      startTime: entry?.startTime.toFixed(2),
      entry,
    } as IMetrics;
    this.metrics.set(metricsName.LCP, metrics);
  };
  getLCP(entryHandler);
};

首次输入延迟（FID）
FID 是从用户第一次与页面交互（例如当他们单击链接、点按按钮或使用由 JavaScript 驱动的自定义控件）直到浏览器对交互作出响应，并实际能够开始处理事件处理程序所经过的时间。
通常来说，我们可以认为，FID 时间在 100ms 内的能 让用户得到良好的使用体验


我们可以手动写一个 FID 获取：

js 代码解读复制代码new PerformanceObserver((entryList) => {
  const entries = entryList.getEntries();
  const entry = entries[entries.length - 1];
  const delay = entry.processingStart - entry.startTime;
  console.log('FID:', delay, entry);
}).observe({ type: 'first-input', buffered: true });


也可以使用 google 的 web-vitals ：

js 代码解读复制代码import { getFID } from 'web-vitals';
// 当 FID 可用时立即进行测量和记录。
getFID(console.log);


还可以 简单封装一下 以适合上文的整体封装

js 代码解读复制代码// 获取 FID
export const getFID = (entryHandler: PerformanceEntryHandler): PerformanceObserver | undefined => {
  return observe('first-input', entryHandler);
};

// 初始化 FID 的获取 及返回
initFID = (): void => {
  const entryHandler = (entry: PerformanceEventTiming) => {
    const metrics = {
      delay: entry.processingStart - entry.startTime,
      entry,
    } as IMetrics;
    this.metrics.set(metricsName.FID, metrics);
  };
  getFID(entryHandler);
};

累计布局偏移（CLS）
CLS 是测量整个页面生命周期（页面可见性变成隐藏）内发生的所有 意外布局偏移 中最大一的 布局偏移分数。；每当一个已渲染的可见元素的位置从一个可见位置变更到下一个可见位置时，就发生了 布局偏移 。
CLS 会衡量在网页的整个生命周期内发生的所有意外布局偏移的得分总和。
简单点说，就是正在阅读文章时，突然页面上某些内容发生了改变；或者你正要点击一个链接或一个按钮，但在手指落下的瞬间，哟？按钮来了一拨灵性走位，导致你点到了别的东西；可以看下图：

通常来说，我们应该将 CLS 分数控制在 0.1 或以下


我这里就直接呈上适合上文的整体封装的代码了，entryHandler函数内的就是关键计算代码：

ts 代码解读复制代码export interface LayoutShift extends PerformanceEntry {
  value: number;
  hadRecentInput: boolean;
}

// 获取 CLS
export const getCLS = (entryHandler: PerformanceEntryHandler): PerformanceObserver | undefined => {
  return observe('layout-shift', entryHandler);
};

// 初始化 CLS 的获取以及返回
initCLS = (): void => {
  let clsValue = 0;
  let clsEntries = [];

  let sessionValue = 0;
  let sessionEntries: Array<LayoutShift> = [];

  const entryHandler = (entry: LayoutShift) => {
    if (!entry.hadRecentInput) {
      const firstSessionEntry = sessionEntries[0];
      const lastSessionEntry = sessionEntries[sessionEntries.length - 1];

      // 如果条目与上一条目的相隔时间小于 1 秒且
      // 与会话中第一个条目的相隔时间小于 5 秒，那么将条目
      // 包含在当前会话中。否则，开始一个新会话。
      if (
        sessionValue &&
        entry.startTime - lastSessionEntry.startTime < 1000 &&
        entry.startTime - firstSessionEntry.startTime < 5000
      ) {
        sessionValue += entry.value;
        sessionEntries.push(entry);
      } else {
        sessionValue = entry.value;
        sessionEntries = [entry];
      }

      // 如果当前会话值大于当前 CLS 值，
      // 那么更新 CLS 及其相关条目。
      if (sessionValue > clsValue) {
        clsValue = sessionValue;
        clsEntries = sessionEntries;

        // 记录 CLS 到 Map 里
        const metrics = {
          entry,
          clsValue,
          clsEntries,
        } as IMetrics;
        this.metrics.set(metricsName.CLS, metrics);
      }
    }
  };
  getCLS(entryHandler);
};


也可以使用 google 的 web-vitals ：

js 代码解读复制代码import {getCLS} from 'web-vitals';
// 在所有需要汇报 CLS 的情况下
// 对其进行测量和记录。
getCLS(console.log);

以技术为中心的性能指标

什么叫以技术为中心的性能指标呢？
我们再来看上面这张之前放过的图，这是 W3C Performance Timeline Level 2 的模型图，图中很多的时间点、时间段，对于用户来说或许并不需要知道，但是 对于技术人员来说 ，采集其中有意义的时间段，做成瀑图，可以让我们从精确数据的角度对网站的性能有一个定义，有一个优化的方向；
关键时间点









































字段描述计算公式备注FP白屏时间responseEnd - fetchStart从请求开始到浏览器开始解析第一批HTML文档字节的时间。TTI首次可交互时间domInteractive - fetchStart浏览器完成所有HTML解析并且完成DOM构建，此时浏览器开始加载资源。DomReadyHTML加载完成时间也就是 DOM Ready 时间。domContentLoadEventEnd - fetchStart单页面客户端渲染下，为生成模板dom树所花费时间；非单页面或单页面服务端渲染下，为生成实际dom树所花费时间'Load页面完全加载时间loadEventStart - fetchStartLoad=首次渲染时间+DOM解析耗时+同步JS执行+资源加载耗时。FirstByte首包时间responseStart - domainLookupStart从DNS解析到响应返回给浏览器第一个字节的时间
关键时间段





















































字段描述计算公式备注DNSDNS查询耗时domainLookupEnd - domainLookupStart如果使用长连接或本地缓存，则数值为0TCPTCP连接耗时connectEnd - connectStart如果使用长连接或本地缓存，则数值为0SSLSSL安全连接耗时connectEnd - secureConnectionStart只在HTTPS下有效，判断secureConnectionStart的值是否大于0,如果为0，转为减connectEndTTFB请求响应耗时responseStart - requestStartTTFB有多种计算方式，相减的参数可以是 requestStart 或者 startTimeTrans内容传输耗时responseEnd - responseStart无DOMDOM解析耗时domInteractive - responseEnd无Res资源加载耗时loadEventStart - domContentLoadedEventEnd表示页面中的同步加载资源。

采集了上述的关键时间段后，我们可以做出一次页面加载的具体性能瀑图，我们可以根据这个图分析性能优化的方向；


简单封装
js 代码解读复制代码export interface MPerformanceNavigationTiming {
  FP?: number;
  TTI?: number;
  DomReady?: number;
  Load?: number;
  FirstByte?: number;
  DNS?: number;
  TCP?: number;
  SSL?: number;
  TTFB?: number;
  Trans?: number;
  DomParse?: number;
  Res?: number;
}

// 获取 NT
const getNavigationTiming = (): MPerformanceNavigationTiming | undefined => {
  const resolveNavigationTiming = (entry: PerformanceNavigationTiming): MPerformanceNavigationTiming => {
    const {
      domainLookupStart,
      domainLookupEnd,
      connectStart,
      connectEnd,
      secureConnectionStart,
      requestStart,
      responseStart,
      responseEnd,
      domInteractive,
      domContentLoadedEventEnd,
      loadEventStart,
      fetchStart,
    } = entry;

    return {
      // 关键时间点
      FP: responseEnd - fetchStart,
      TTI: domInteractive - fetchStart,
      DomReady: domContentLoadedEventEnd - fetchStart,
      Load: loadEventStart - fetchStart,
      FirstByte: responseStart - domainLookupStart,
      // 关键时间段
      DNS: domainLookupEnd - domainLookupStart,
      TCP: connectEnd - connectStart,
      SSL: secureConnectionStart ? connectEnd - secureConnectionStart : 0,
      TTFB: responseStart - requestStart,
      Trans: responseEnd - responseStart,
      DomParse: domInteractive - responseEnd,
      Res: loadEventStart - domContentLoadedEventEnd,
    };
  };

  const navigation =
    // W3C Level2  PerformanceNavigationTiming
    // 使用了High-Resolution Time，时间精度可以达毫秒的小数点好几位。
    performance.getEntriesByType('navigation').length > 0
      ? performance.getEntriesByType('navigation')[0]
      : performance.timing; // W3C Level1  (目前兼容性高，仍然可使用，未来可能被废弃)。
  return resolveNavigationTiming(navigation as PerformanceNavigationTiming);
};

// 初始化 NT 的获取以及返回
initNavigationTiming = (): void => {
  const navigationTiming = getNavigationTiming();
  const metrics = navigationTiming as IMetrics;
  this.metrics.set(metricsName.NT, metrics);
};

其它也有意义的指标
静态资源加载

我们可以获取每次加载时所访问的静态资源，将收集到的静态资源做成瀑图等分析图形，来找出导致静态资源加载时间过长的问题所在。

简单写一个 静态资源的获取：

js 代码解读复制代码const resource = performance.getEntriesByType('resource')
const formatResourceArray = resource.map(item => {
  return {
    name: item.name,                    //资源地址
    startTime: item.startTime,          //开始时间
    responseEnd: item.responseEnd,      //结束时间
    time: item.duration,                //消耗时间
    initiatorType: item.initiatorType, //资源类型
    transferSize: item.transferSize,    //传输大小
    //请求响应耗时 ttfb = item.responseStart - item.startTime
    //内容下载耗时 tran = item.responseEnd - item.responseStart 
    //但是受到跨域资源影响。除非资源设置允许获取timing
  };
});


还可以封装一下以适合上文的整体封装

ts 代码解读复制代码export interface ResourceFlowTiming {
  name: string;
  transferSize: number;
  initiatorType: string;
  startTime: number;
  responseEnd: number;
  dnsLookup: number;
  initialConnect: number;
  ssl: number;
  request: number;
  ttfb: number;
  contentDownload: number;
}

// 获取 RF
export const getResourceFlow = (resourceFlow: Array<ResourceFlowTiming>): PerformanceObserver | undefined => {
  const entryHandler = (entry: PerformanceResourceTiming) => {
    const {
      name,
      transferSize,
      initiatorType,
      startTime,
      responseEnd,
      domainLookupEnd,
      domainLookupStart,
      connectStart,
      connectEnd,
      secureConnectionStart,
      responseStart,
      requestStart,
    } = entry;
    resourceFlow.push({
      // name 资源地址
      name,
      // transferSize 传输大小
      transferSize,
      // initiatorType 资源类型
      initiatorType,
      // startTime 开始时间
      startTime,
      // responseEnd 结束时间
      responseEnd,
      // 贴近 Chrome 的近似分析方案，受到跨域资源影响
      dnsLookup: domainLookupEnd - domainLookupStart,
      initialConnect: connectEnd - connectStart,
      ssl: connectEnd - secureConnectionStart,
      request: responseStart - requestStart,
      ttfb: responseStart - requestStart,
      contentDownload: responseStart - requestStart,
    });
  };

  return observe('resource', entryHandler);
};

// 初始化 RF 的获取以及返回
initResourceFlow = (): void => {
  const resourceFlow: Array<ResourceFlowTiming> = [];
  const resObserve = getResourceFlow(resourceFlow);

  const stopListening = () => {
    if (resObserve) {
      resObserve.disconnect();
    }
    const metrics = resourceFlow as IMetrics;
    this.metrics.set(metricsName.RF, metrics);
  };
  // 当页面 pageshow 触发时，中止
  window.addEventListener('pageshow', stopListening, { once: true, capture: true });
};

静态资源加载的缓存命中率
很多的一些资源，比如 img图片等，在用户加载后这些资源就会被缓存起来，再下一次进入时判断缓存类型、是否过期来决定是否使用缓存；那么我们就可以统计每一次用户进入时的一个缓存命中率；
那么，如何判断用户的资源是否命中了缓存呢？，其实很简单，如果静态资源被缓存了，它具有以下两个特征：

静态资源的 duration 为0；
静态资源的 transferSize 不为0；

根据上面这两个特征，我们就可以计算每次加载的缓存命中率：
js 代码解读复制代码const resource = performance.getEntriesByType('resource');
let cacheQuantity = 0;
const formatResourceArray = resource.map((item) => {
  if (item.duration == 0 && item.transferSize !== 0) cacheQuantity++;
  return {
    name: item.name, //资源地址
    startTime: item.startTime, //开始时间
    responseEnd: item.responseEnd, //结束时间
    time: item.duration, //消耗时间
    initiatorType: item.initiatorType, //资源类型
    transferSize: item.transferSize, //传输大小
    //请求响应耗时 ttfb = item.responseStart - item.startTime
    //内容下载耗时 tran = item.responseEnd - item.responseStart
    //但是受到跨域资源影响。除非资源设置允许获取timing
  };
});
console.log('缓存命中率', (cacheQuantity / resource.length).toFixed(2));

PS：跨域资源(CDN)
获取页面资源时间详情时，有跨域的限制。默认情况下，跨域资源以下属性会被设置为 0

redirectStart
redirectEnd
domainLookupStart
domainLookupEnd
connectStart
connectEnd
secureConnectionStart
requestStart
responseStart
transferSize

如果想获取资源的具体时间，跨域资源需要设置响应头 Timing-Allow-Origin

对于可控跨域资源例如自家 CDN，Timing-Allow-Origin 的响应头 origins 至少得设置了主页面的域名，允许获取资源时间
一般对外公共资源设置为 Timing-Allow-Origin: *。

分析得出的指标
分析得出的指标，意味着不是一次采集就能得到的指标数据，而指的是对于整个应用来说，对采集上来的众多数据进行分析而得出的指标情况；

首次加载跳出率：第一个页面完全加载前用户跳出率；
慢开比：完全加载耗时超过5s的PV占比；
多维度分析：对地域、网络、页面等多个维度下的性能情况；
