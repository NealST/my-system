# Norms To Judge Web Page Performance

When we talk about the performace of web pages, there is a question that how we can judge a good perfomance case or not. In an objective way, we need some measurable norms as the judgement standard. In general, there are main two parts of norms, including load experience and interaction experience. Each part has some concret norms to help us measure the performance of web page.

## Load Experience
In load process, we have main four norms to analyze the performance, including first-paint, first-screen-paint, first-contentful-paint and largest-contentful-paint. All the definition of these norms are from the point of user.

### First-Paint(named as FP)

#### Definition
The gap time between starting request page and the first pixel content shows on the screen.

#### How to get
```javascript
function getFirstPaint() {
  let firstPaints = {};
  if (typeof performance.getEntriesByType === 'function') {
    let performanceEntries = performance.getEntriesByType('paint') || [];
    performanceEntries.forEach((entry) => {
      if (entry.name === 'first-paint') {
        firstPaints.firstPaint = entry.startTime;
      }
    });
  } else {
    if (chrome && chrome.loadTimes) {
      let loadTimes = window.chrome.loadTimes();
      let {firstPaintTime, startLoadTime} = loadTimes;
      firstPaints.firstPaint = (firstPaintTime - startLoadTime) * 1000;
    } else if (performance.timing && typeof performance.timing.msFirstPaint === 'number') {
      let {msFirstPaint, navigationStart} = performance.timing;
      firstPaints.firstPaint = msFirstPaint - navigationStart;
    }
  }
  return firstPaints;
}
```

### First Screen Paint(named as FSP)

#### Definition
The gap time between starting request page and all the first screen content are painted on the page(including images are loaded and rendered) ,the page is displayed stable for users.

#### How to get

### First Contentful Paint(named as FCP)

#### Definition

The gap time between starting request page and any content of the page shows on the screen. Specifically, the content includes text, images, svg and non white canvas.

#### How to get
```javascript
function getContentfulPaint() {
  const firstPaints = {};
  if (typeof performance.getEntriesByType === 'function') {
    let performanceEntries = performance.getEntriesByType('paint') || [];
    performanceEntries.forEach((entry) => {
      if (entry.name === 'first-contentful-paint') {
        firstPaints.firstContentfulPaint = entry.startTime;
      }
    });
  }
}
```

#### Good standard
![](https://web.dev/static/articles/fcp/image/good-fcp-values-18.svg?hl=zh-cn)

### Largest Contentful Paint(named as LCP)

#### Definition
The gap time between starting request page and the largest content shows on the screen

#### How to get
you could make the web-vitals library as accordance, check [onLCP](https://github.com/GoogleChrome/web-vitals/blob/main/src/onLCP.ts)

#### Good standard
![](https://web.dev/static/articles/lcp/image/good-lcp-values.svg?hl=zh-cn)

## Interaction Experience

### Time to Interactive(named as TTI)

#### Definition
The gap time between starting request page and the page is ready to be interactive. This norm has not been recommended by google

#### How to get
you could make [tti-polyfill](https://www.npmjs.com/package/tti-polyfill) library as accordance

### First Input Delay(named as FID)

#### Definition
The gap time between the first time user interact with your page and your page make reaction to this interaction. We can use this picture to understand what is FID

![](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/161/1611297584161-11ef972c-d5b5-4853-9dd2-f57b4d96252b.png)

#### How to get
you could make the web-vitals library as accordance, check [onFID](https://github.com/GoogleChrome/web-vitals/blob/main/src/onFID.ts)

#### Good standard

![](https://web.dev/static/articles/fid/image/good-fid-values-25.svg?hl=zh-cn)

### Total Blocking Time(named as TBT)

#### Definition
The total time that users' interaction is blocked from FCP to TTI, when the main thread runs last more than 50ms, we can say that the page is in no response state.

For example,you have five tasks, like this:
![](https://web.dev/static/articles/tbt/image/a-tasks-timeline-the-mai-72582d2e9e8a8.svg?hl=zh-cn)

among these tasks,there are three long time task that runs more than 50ms, following picture shows the block time in every task.
![](https://web.dev/static/articles/tbt/image/a-tasks-timeline-the-mai-58cb4f9bb2cf3.svg?hl=zh-cn)

#### How to get
you can use lighthouse to detect this norm.

### Frames Per Second(named as FPS)

#### Definition
The total frames that can be painted every second. If this norm is less than 60, users will get a perception of slowness.

#### How to get
```javascript
var lastTime = performance.now();
var frame = 0;
var lastFameTime = performance.now();

var loop = function(time) {
	var now =  performance.now();
	var fs = (now - lastFameTime);
	lastFameTime = now;
	var fps = Math.round(1000/fs);
	frame++;
	if (now > 1000 + lastTime) {
		var fps = Math.round( ( frame * 1000 ) / ( now - lastTime ) );
		frame = 0;    
		lastTime = now;    
	};           
	window.requestAnimationFrame(loop);   
}
```
