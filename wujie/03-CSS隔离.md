# 无界微前端源码解析：CSS 隔离

> 深入分析 Shadow DOM 实现 CSS 隔离的原理和样式处理机制。

## Shadow DOM 基础

Shadow DOM 是 Web Components 的核心技术，提供原生的样式隔离：

```javascript
// 创建 Shadow DOM
const host = document.createElement('div');
const shadowRoot = host.attachShadow({ mode: 'open' });
shadowRoot.innerHTML = `
  <style>
    p { color: red; }  /* 只影响 Shadow DOM 内部 */
  </style>
  <p>Hello Shadow DOM</p>
`;
```

## Web Component 定义

```typescript
// packages/wujie-core/src/shadow.ts
export function defineWujieWebComponent() {
  const customElements = window.customElements;
  
  if (customElements && !customElements?.get("wujie-app")) {
    class WujieApp extends HTMLElement {
      // 元素插入 DOM 时触发
      connectedCallback(): void {
        if (this.shadowRoot) return;
        
        // 创建 Shadow DOM
        const shadowRoot = this.attachShadow({ mode: "open" });
        
        // 获取沙箱实例
        const sandbox = getWujieById(this.getAttribute(WUJIE_APP_ID));
        
        // patch 元素效果
        patchElementEffect(shadowRoot, sandbox.iframe.contentWindow);
        
        // 关联 shadowRoot
        sandbox.shadowRoot = shadowRoot;
      }

      // 元素从 DOM 移除时触发
      disconnectedCallback(): void {
        const sandbox = getWujieById(this.getAttribute(WUJIE_APP_ID));
        sandbox?.unmount();
      }
    }
    
    customElements?.define("wujie-app", WujieApp);
  }
}
```

## 创建 Web Component

```typescript
// packages/wujie-core/src/shadow.ts
export function createWujieWebComponent(id: string): HTMLElement {
  const contentElement = window.document.createElement("wujie-app");
  contentElement.setAttribute(WUJIE_APP_ID, id);
  contentElement.classList.add(WUJIE_IFRAME_CLASS);
  return contentElement;
}
```

使用时：

```typescript
// packages/wujie-core/src/sandbox.ts - active 方法
if (this.shadowRoot) {
  this.el = renderElementToContainer(this.shadowRoot.host, el);
} else {
  // 创建 Web Component 并插入容器
  this.el = renderElementToContainer(createWujieWebComponent(this.id), el);
}
```

## 渲染到 Shadow DOM

```typescript
// packages/wujie-core/src/shadow.ts
export async function renderTemplateToShadowRoot(
  shadowRoot: ShadowRoot,
  iframeWindow: Window,
  template: string
): Promise<void> {
  // 1. 将 template 转换为 HTML 元素
  const html = renderTemplateToHtml(iframeWindow, template);
  
  // 2. 处理 css-before-loader 和 css-after-loader
  const processedHtml = await processCssLoaderForTemplate(iframeWindow.__WUJIE, html);
  
  // 3. 插入 Shadow DOM
  shadowRoot.appendChild(processedHtml);
  
  // 4. 添加遮罩层（防止样式闪烁）
  const shade = document.createElement("div");
  shade.setAttribute("style", WUJIE_SHADE_STYLE);
  processedHtml.insertBefore(shade, processedHtml.firstChild);
  
  // 5. 设置 head 和 body 引用
  shadowRoot.head = shadowRoot.querySelector("head");
  shadowRoot.body = shadowRoot.querySelector("body");

  // 6. 修复 html parentNode
  Object.defineProperty(shadowRoot.firstChild, "parentNode", {
    enumerable: true,
    configurable: true,
    get: () => iframeWindow.document,
  });

  // 7. patch 渲染效果
  patchRenderEffect(shadowRoot, iframeWindow.__WUJIE.id, false);
}
```

## 模板转 HTML

```typescript
// packages/wujie-core/src/shadow.ts
function renderTemplateToHtml(iframeWindow: Window, template: string): HTMLHtmlElement {
  const sandbox = iframeWindow.__WUJIE;
  const { head, body, alive, execFlag } = sandbox;
  const document = iframeWindow.document;
  
  // 1. 解析 HTML
  const parser = new DOMParser();
  const parsedDocument = parser.parseFromString(template, "text/html");
  const parsedHtml = parsedDocument.documentElement as HTMLHtmlElement;
  
  // 2. 创建 HTML 元素
  let html = document.createElement("html");
  html.innerHTML = template;
  
  // 3. 复制属性
  const sourceAttributes = parsedHtml.attributes;
  for (let i = 0; i < sourceAttributes.length; i++) {
    html.setAttribute(sourceAttributes[i].name, sourceAttributes[i].value);
  }
  
  // 4. 复用 head 和 body（保活场景）
  if (!alive && execFlag) {
    html = replaceHeadAndBody(html, head, body);
  } else {
    sandbox.head = html.querySelector("head");
    sandbox.body = html.querySelector("body");
  }
  
  // 5. patch 所有元素
  const ElementIterator = document.createTreeWalker(html, NodeFilter.SHOW_ELEMENT, null, false);
  let nextElement = ElementIterator.currentNode as HTMLElement;
  while (nextElement) {
    patchElementEffect(nextElement, iframeWindow);
    
    // 处理相对路径
    const relativeAttr = relativeElementTagAttrMap[nextElement.tagName];
    const url = nextElement[relativeAttr];
    if (relativeAttr) {
      nextElement.setAttribute(relativeAttr, getAbsolutePath(url, nextElement.baseURI || ""));
    }
    
    nextElement = ElementIterator.nextNode() as HTMLElement;
  }
  
  return html;
}
```

## CSS Loader 处理

```typescript
// packages/wujie-core/src/shadow.ts
async function processCssLoaderForTemplate(sandbox: Wujie, html: HTMLHtmlElement): Promise<HTMLHtmlElement> {
  const { plugins, replace, proxyLocation } = sandbox;
  const cssLoader = getCssLoader({ plugins, replace });
  const cssBeforeLoaders = getPresetLoaders("cssBeforeLoaders", plugins);
  const cssAfterLoaders = getPresetLoaders("cssAfterLoaders", plugins);
  const curUrl = getCurUrl(proxyLocation);

  return await Promise.all([
    // 处理 cssBeforeLoaders（插入到最前面）
    Promise.all(
      getExternalStyleSheets(cssBeforeLoaders, sandbox.fetch, sandbox.lifecycles.loadError)
        .map(({ src, contentPromise }) => contentPromise.then((content) => ({ src, content })))
    ).then((contentList) => {
      contentList.forEach(({ src, content }) => {
        if (!content) return;
        const styleElement = document.createElement("style");
        styleElement.appendChild(document.createTextNode(cssLoader(content, src, curUrl)));
        html.insertBefore(styleElement, html.querySelector("head") || html.firstChild);
      });
    }),
    
    // 处理 cssAfterLoaders（插入到最后面）
    Promise.all(
      getExternalStyleSheets(cssAfterLoaders, sandbox.fetch, sandbox.lifecycles.loadError)
        .map(({ src, contentPromise }) => contentPromise.then((content) => ({ src, content })))
    ).then((contentList) => {
      contentList.forEach(({ src, content }) => {
        if (!content) return;
        const styleElement = document.createElement("style");
        styleElement.appendChild(document.createTextNode(cssLoader(content, src, curUrl)));
        html.appendChild(styleElement);
      });
    }),
  ]).then(() => html);
}
```

## CSS 相对路径处理

```typescript
// packages/wujie-core/src/plugin.ts
function cssRelativePathResolve(code: string, src: string, base: string) {
  const baseUrl = src ? getAbsolutePath(src, base) : base;
  
  // 匹配 url() 中的路径
  const urlReg = /url\((['"]?)((?:[^()]+|\((?:[^()]+|\([^()]*\))*\))*)(\1)\)/g;

  return code.replace(urlReg, (_m, pre, url, post) => {
    // 跳过 base64
    if (/^data:/.test(url)) {
      return _m;
    }
    // 转换为绝对路径
    return `url(${pre}${getAbsolutePath(url, baseUrl)}${post})`;
  });
}

// 默认插件
const defaultPlugin = {
  cssLoader: cssRelativePathResolve,
  cssBeforeLoaders: [{ content: "html {view-transition-name: none;}" }],
};
```

## :root 样式处理

Shadow DOM 中 `:root` 选择器不生效，需要转换为 `:host`：

```typescript
// packages/wujie-core/src/shadow.ts
const cssSelectorMap = {
  ":root": ":host",
};

export function getPatchStyleElements(rootStyleSheets: Array<CSSStyleSheet>): Array<HTMLStyleElement | null> {
  const rootCssRules = [];
  const fontCssRules = [];
  const rootStyleReg = /:root/g;

  // 遍历所有样式表
  for (let i = 0; i < rootStyleSheets.length; i++) {
    const cssRules = rootStyleSheets[i]?.cssRules ?? [];
    for (let j = 0; j < cssRules.length; j++) {
      const cssRuleText = cssRules[j].cssText;
      
      // :root 样式转换为 :host
      if (rootStyleReg.test(cssRuleText)) {
        rootCssRules.push(cssRuleText.replace(rootStyleReg, (match) => cssSelectorMap[match]));
      }
      
      // @font-face 需要提取到外部
      if (cssRules[j].type === CSSRule.FONT_FACE_RULE) {
        fontCssRules.push(cssRuleText);
      }
    }
  }

  let rootStyleSheetElement = null;
  let fontStyleSheetElement = null;

  // 创建 :host 样式
  if (rootCssRules.length) {
    rootStyleSheetElement = window.document.createElement("style");
    rootStyleSheetElement.innerHTML = rootCssRules.join("");
  }

  // 创建 @font-face 样式（放到 Shadow DOM 外部）
  if (fontCssRules.length) {
    fontStyleSheetElement = window.document.createElement("style");
    fontStyleSheetElement.innerHTML = fontCssRules.join("");
  }

  return [rootStyleSheetElement, fontStyleSheetElement];
}
```

应用补丁：

```typescript
// packages/wujie-core/src/sandbox.ts
public patchCssRules(): void {
  if (this.degrade) return;
  if (this.shadowRoot.host.hasAttribute(WUJIE_DATA_ATTACH_CSS_FLAG)) return;
  
  const [hostStyleSheetElement, fontStyleSheetElement] = getPatchStyleElements(
    Array.from(this.iframe.contentDocument.querySelectorAll("style"))
      .map((styleSheetElement) => styleSheetElement.sheet)
  );
  
  // :host 样式插入 Shadow DOM head
  if (hostStyleSheetElement) {
    this.shadowRoot.head.appendChild(hostStyleSheetElement);
    this.styleSheetElements.push(hostStyleSheetElement);
  }
  
  // @font-face 样式插入 Shadow DOM 外部
  if (fontStyleSheetElement) {
    this.shadowRoot.host.appendChild(fontStyleSheetElement);
  }
  
  // 标记已处理
  (hostStyleSheetElement || fontStyleSheetElement) &&
    this.shadowRoot.host.setAttribute(WUJIE_DATA_ATTACH_CSS_FLAG, "");
}
```

## 样式重建

子应用重新激活时，需要重建样式：

```typescript
// packages/wujie-core/src/sandbox.ts
public rebuildStyleSheets(): void {
  if (this.styleSheetElements && this.styleSheetElements.length) {
    this.styleSheetElements.forEach((styleSheetElement) => {
      rawElementAppendChild.call(
        this.degrade ? this.document.head : this.shadowRoot.head,
        styleSheetElement
      );
    });
  }
  this.patchCssRules();
}
```

## 降级模式

不支持 Shadow DOM 时，使用 iframe 渲染：

```typescript
// packages/wujie-core/src/shadow.ts
export async function renderTemplateToIframe(
  renderDocument: Document,
  iframeWindow: Window,
  template: string
): Promise<void> {
  // 1. 转换模板
  const html = renderTemplateToHtml(iframeWindow, template);
  
  // 2. 处理 CSS loader
  const processedHtml = await processCssLoaderForTemplate(iframeWindow.__WUJIE, html);
  
  // 3. 替换文档
  renderDocument.replaceChild(processedHtml, renderDocument.documentElement);

  // 4. 修复 parentNode
  Object.defineProperty(renderDocument.documentElement, "parentNode", {
    enumerable: true,
    configurable: true,
    get: () => iframeWindow.document,
  });

  // 5. patch 渲染效果
  patchRenderEffect(renderDocument, iframeWindow.__WUJIE.id, true);
}
```

## 小结

无界的 CSS 隔离机制：

1. **Shadow DOM**：原生样式隔离，子应用样式不会影响主应用
2. **Web Component**：封装 Shadow DOM，提供生命周期钩子
3. **CSS Loader**：处理相对路径、注入前置/后置样式
4. **:root 转换**：将 `:root` 转换为 `:host` 适配 Shadow DOM
5. **@font-face 提取**：字体定义需要放到 Shadow DOM 外部才能生效
6. **降级方案**：不支持时使用 iframe 渲染

下一篇我们将分析 JS 隔离的 Proxy 代理机制。

---

> 📦 源码版本：wujie v1.0.22
>
> 上一篇：[沙箱机制](https://juejin.cn/post/7588704463621455891)
>
> 下一篇：[JS 隔离](https://juejin.cn/post/7588570963295633451)
