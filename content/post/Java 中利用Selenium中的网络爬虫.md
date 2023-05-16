---
categories:
    - dev
date: "2023-05-16T22:35:43+08:00"
description: ""
image: ""
slug: java-zhong-li-yong-seleniumzhong-de-wang-luo-pa-chong
tags:
    - java
    - data
title: Java 中利用Selenium中的网络爬虫
---


这里第一部分主要说明下如何利用 Selenium 来爬虫，虽然 Selenium 这个组件相比于直接发送 API 请求的方式很重，但是实际情况来说，也只有 Selenium 足够应付各种复杂的网络情况。虽然效率不高，但是我们爬虫更关注是数据能够能够通过程序自动的获取。由于需要考虑到与 SpringBoot 的集成，这里考虑使用Java版本的来获取网络数据。

## Selenium 的入门

下面的版本，基于JDK17 和 Selenium 4 的版本。

1. 获得依赖，Gradle 的依赖如下。 

   ```groovy
   # 下面的配置仅在在测试环境中使用
   testImplementation 'org.seleniumhq.selenium:selenium-java:4.9.1'
   ```

2. 安装浏览驱动，官网指定了四种方式，这里使用最后一种，代码中硬编码的方式。[Install browser drivers](https://www.selenium.dev/documentation/webdriver/getting_started/install_drivers/)

   ```java
   System.setProperty("webdriver.chrome.driver","/path/to/chromedriver");
   ChromeDriver driver = new ChromeDriver();
   ```

   各个操作系统对应的浏览器驱动下载地址

   | Browser           | Supported OS                | Maintained by    | Download                                                     | Issue Tracker                                                |
   | ----------------- | --------------------------- | ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
   | Chromium/Chrome   | Windows/macOS/Linux         | Google           | [Downloads](https://chromedriver.chromium.org/downloads)     | [Issues](https://bugs.chromium.org/p/chromedriver/issues/list) |
   | Firefox           | Windows/macOS/Linux         | Mozilla          | [Downloads](https://github.com/mozilla/geckodriver/releases) | [Issues](https://github.com/mozilla/geckodriver/issues)      |
   | Edge              | Windows/macOS/Linux         | Microsoft        | [Downloads](https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/) | [Issues](https://github.com/MicrosoftEdge/EdgeWebDriver/issues) |
   | Internet Explorer | Windows                     | Selenium Project | [Downloads](https://www.selenium.dev/downloads)              | [Issues](https://github.com/SeleniumHQ/selenium/labels/D-IE) |
   | Safari            | macOS High Sierra and newer | Apple            | Built in                                                     | [Issues](https://bugreport.apple.com/logon)                  |

   Note: The Opera driver no longer works with the latest functionality of Selenium and is currently officially unsupported.

3. （可选） 兼容性检查

   比如，我是用chrome浏览器，我需要先检查浏览器的版本。 在chrome中输入。 `chrome://version` 信息，可以看到类似下面的版本信息。主要检查Google Chrome的版本同 Driver 的驱动设置是不是匹配。

   ```text
   Google Chrome	113.0.5672.92 (Official Build) (arm64) 
   Revision	b6f521170062a1fa8a82c33fb223b06fec566da1-refs/branch-heads/5672_63@{#10}
   OS	macOS Version 12.1 (Build 21C52)
   JavaScript	V8 11.3.244.8
   User Agent	Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/113.0.0.0 Safari/537.36
   Command Line	/Applications/Google Chrome.app/Contents/MacOS/Google Chrome --flag-switches-begin --flag-switches-end
   Executable Path	/Applications/Google Chrome.app/Contents/MacOS/Google Chrome
   Profile Path	/Users/jinghui.zhang/Library/Application Support/Google/Chrome/Default
   Linker	lld
   Active Variations	50eaf59c-ca7d8d80
   .....
   ```

#### 编写脚本

在安装好依赖和浏览器驱动之后，开始编写脚本

1. 启动其中的 Session。更多的关于 Selenium 中session概念，可以参考 [driver sessions](https://www.selenium.dev/documentation/webdriver/drivers/)。

   ```java
   WebDriver driver = new ChromeDriver();
   ```

2. 在浏览器中做一些动作。比如，我们做一些导航到某个网站的操作。[navigating](https://www.selenium.dev/documentation/webdriver/interactions/navigation/) 信息。

   ```java
   driver.get("https://www.selenium.dev/selenium/web/web-form.html");
   ```

3. 获得浏览器中的一些信息，比如，网站标题。更多的[information about the browser](https://www.selenium.dev/documentation/webdriver/interactions/) 

   ```java
   String title = driver.getTitle();
   ```

4. 设置浏览器的等待策略。也就是网页刷新的到什么时候才是准备好的。设置一个等待时间，虽然不是最好的解决方案，但是确实一个最容易理解的方案。更多的[Waiting strategies](https://www.selenium.dev/documentation/webdriver/waits/).

   ```java
   driver.manage().timeouts().implicitlyWait(Duration.ofMillis(500));
   ```

5. 查找页面上的元素。更多的[finding an element](https://www.selenium.dev/documentation/webdriver/elements/)

   ```java
   WebElement textBox = driver.findElement(By.name("my-text"));
   WebElement submitButton = driver.findElement(By.cssSelector("button"));
   ```

6. 对找到的元素做一些动作。更多的 [actions to take on an element](https://www.selenium.dev/documentation/webdriver/elements/interactions/)

   ```java
   textBox.sendKeys("Selenium");
   submitButton.click();
   ```

7. 请求一个元素信息。更多的[information that can be requested](https://www.selenium.dev/documentation/webdriver/elements/information/).

   ```java
   String value = message.getText();
   ```

8. 不用了记得关闭Session

   ```java
   driver.quit();
   ```

完整的[源代码](https://github.com/SeleniumHQ/seleniumhq.github.io/blob/trunk/examples/java/src/test/java/dev/selenium/getting_started/FirstScriptTest.java)设置。

## Driver Sessions

Driver Session就是启动或者关闭一个浏览器。

## [Creating Sessions ](https://www.selenium.dev/documentation/webdriver/drivers/#creating-sessions)

创建一个新的Session，Java中都可以通过class 来创建对象，并且给出一些参数来创建Session。

- [Options](https://www.selenium.dev/documentation/webdriver/drivers/options/) 的配置可以对这个Session做一些设置。
- Some form of [CommandExecutor](https://www.selenium.dev/documentation/webdriver/drivers/executors/) (the implementation varies between languages)
- [Listeners](https://www.selenium.dev/documentation/webdriver/drivers/listeners/)

具有 Local Driver 和 Remote Driver 两种驱动类型。

> 建议始终使用quit 方法来关闭Session。

更多章节内容如下：

1. [Browser Options](https://www.selenium.dev/documentation/webdriver/drivers/options/)
2. [Command executors](https://www.selenium.dev/documentation/webdriver/drivers/executors/)
3. [Command Listeners](https://www.selenium.dev/documentation/webdriver/drivers/listeners/)
4. [Browser Service](https://www.selenium.dev/documentation/webdriver/drivers/service/)
5. [Remote WebDriver](https://www.selenium.dev/documentation/webdriver/drivers/remote_webdriver/)

## [Supported Browsers | Selenium](https://www.selenium.dev/documentation/webdriver/browsers/)

[Chrome specific functionality](https://www.selenium.dev/documentation/webdriver/browsers/chrome/)

These are capabilities and features specific to Google Chrome browsers.

[Edge specific functionality](https://www.selenium.dev/documentation/webdriver/browsers/edge/)

These are capabilities and features specific to Microsoft Edge browsers.

[Firefox specific functionality](https://www.selenium.dev/documentation/webdriver/browsers/firefox/)

These are capabilities and features specific to Mozilla Firefox browsers.

[IE specific functionality](https://www.selenium.dev/documentation/webdriver/browsers/internet_explorer/)

These are capabilities and features specific to Microsoft Internet Explorer browsers.

[Safari specific functionality](https://www.selenium.dev/documentation/webdriver/browsers/safari/)

These are capabilities and features specific to Apple Safari browsers.

## [Waits](https://www.selenium.dev/documentation/webdriver/waits/)

## [Web elements](https://www.selenium.dev/documentation/webdriver/elements/)

Identifying and working with element objects in the DOM. The majority of most people’s Selenium code involves working with web elements.

------

1. [File Upload](https://www.selenium.dev/documentation/webdriver/elements/file_upload/)

2. [Locator strategies](https://www.selenium.dev/documentation/webdriver/elements/locators/) Ways to identify one or more specific elements in the DOM.

3. [Finding web elements](https://www.selenium.dev/documentation/webdriver/elements/finders/) Locating the elements based on the provided locator values.

4. [Interacting with web elements](https://www.selenium.dev/documentation/webdriver/elements/interactions/) A high-level instruction set for manipulating form controls.

5. [Information about web elements](https://www.selenium.dev/documentation/webdriver/elements/information/) What you can learn about an element.

### 定位策略

官网[Locator strategies](https://www.selenium.dev/documentation/webdriver/elements/locators/)。

#### 传统的定位方式

Selenium provides support for these 8 traditional location strategies in WebDriver:

| Locator           | Description                                                  |
| ----------------- | ------------------------------------------------------------ |
| class name        | Locates elements whose class name contains the search value (compound class names are not permitted) |
| css selector      | Locates elements matching a CSS selector                     |
| id                | Locates elements whose ID attribute matches the search value |
| name              | Locates elements whose NAME attribute matches the search value |
| link text         | Locates anchor elements whose visible text matches the search value |
| partial link text | Locates anchor elements whose visible text contains the search value. If multiple elements are matching, only the first one will be selected. |
| tag name          | Locates elements whose tag name matches the search value     |
| xpath             | Locates elements matching an XPath expression                |

下面举一个例子说明下，这8种方式的用法。假设渲染后的 HTML 如下：

```html
<html>
<body>
<style>
.information {
  background-color: white;
  color: black;
  padding: 10px;
}
</style>
<h2>Contact Selenium</h2>

<form action="/action_page.php">
  <input type="radio" name="gender" value="m" />Male &nbsp;
  <input type="radio" name="gender" value="f" />Female <br>
  <br>
  <label for="fname">First name:</label><br>
  <input class="information" type="text" id="fname" name="fname" value="Jane"><br><br>
  <label for="lname">Last name:</label><br>
  <input class="information" type="text" id="lname" name="lname" value="Doe"><br><br>
  <label for="newsletter">Newsletter:</label>
  <input type="checkbox" name="newsletter" value="1" /><br><br>
  <input type="submit" value="Submit">
</form> 

<p>To know more about Selenium, visit the official page 
<a href ="www.selenium.dev">Selenium Official Page</a> 
</p>

</body>
</html>
```

这些方式都可以方便我们定位问题

1. 通过类选择器。 HTML 页面 Web 元素可以具有属性类。我们可以在上面显示的 HTML 片段中看到一个示例。我们可以使用 Selenium 中可用的类名定位器来识别这些元素。

   ```java
   WebDriver driver = new ChromeDriver();
   driver.findElement(By.className("information"));  
   ```

2. CSS 选择器。CSS 是用于设置 HTML 页面样式的语言。 我们可以使用 css 选择器定位器策略来识别页面上的元素。 如果元素有一个 id，我们创建定位器为 `css = #id`。 否则我们遵循的格式是 `css =[attribute=value]` 。 让我们从上面的 HTML 片段中看一个例子。 我们将使用 css 为名字文本框创建定位器。

   ```java
   WebDriver driver = new ChromeDriver();
   driver.findElement(By.cssSelector("#fname"));
   ```

3. ID 选择器。我们可以使用网页中元素可用的 ID 属性来定位它。 通常，ID 属性对于网页上的元素应该是唯一的。 我们将使用它来识别姓氏字段。

    ```java
    WebDriver driver = new ChromeDriver();
    driver.findElement(By.id("lname"));
    ```

4. Name选择器。我们可以使用网页中元素可用的 NAME 属性来定位它。 通常 NAME 属性对于网页上的元素应该是唯一的。 我们将使用它来识别 Newsletter 复选框。

    ```java
    WebDriver driver = new ChromeDriver();
    driver.findElement(By.name("newsletter"));  
    ```

5. link text。 如果我们要定位的元素是一个链接，我们可以使用链接文本定位器在网页上识别它。 链接文本是链接显示的文本。 在共享的 HTML 片段中，我们有一个可用的链接，让我们看看如何找到它。

    ```java
    WebDriver driver = new ChromeDriver();
    driver.findElement(By.linkText("Selenium Official Page"));
    ```

6. partial link text。如果我们要定位的元素是一个链接，我们可以使用部分链接文本定位器在网页上识别它。 链接文本是链接显示的文本。 我们可以将部分文本作为值传递。 在共享的 HTML 片段中，我们有一个可用的链接，让我们看看如何找到它。

    ```java
    WebDriver driver = new ChromeDriver();
    driver.findElement(By.partialLinkText("Official Page"));
    ```

7. tag name。我们可以使用 HTML TAG 本身作为定位器来识别页面上的 Web 元素。 从上面共享的 HTML 片段中，让我们使用它的 html 标签“a”来识别链接。

    ```java
    WebDriver driver = new ChromeDriver();
    driver.findElement(By.tagName("a"));
    ```

8. xpath。 一个HTML文档可以看作是一个XML文档，然后我们就可以使用xpath来遍历到达感兴趣元素的路径来定位元素。 XPath 可以是绝对 xpath，它是从文档的根目录创建的。 示例 - `/html/form/input[1]`。 这将返回男性单选按钮。 或者 xpath 可能是相对的。 示例- `//input[@name=‘fname’]`。 这将返回名字文本框。 让我们使用 xpath 为女性单选按钮创建定位器。

    ```java
    WebDriver driver = new ChromeDriver();
    driver.findElement(By.xpath("//input[@value='f']"));
    ```

CSS中的语法定义，可以参考[CSS 选择器 - CSS：层叠样式表 | MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Selectors)。

### 相对定位 Relative Locators

**Selenium 4** introduces Relative Locators (previously called as *Friendly Locators*). These locators are helpful when it is not easy to construct a locator for the desired element, but easy to describe spatially where the element is in relation to an element that does have an easily constructed locator.

#### How it works

Selenium uses the JavaScript function [getBoundingClientRect()](https://developer.mozilla.org/en-US/docs/Web/API/Element/getBoundingClientRect) to determine the size and position of elements on the page, and can use this information to locate neighboring elements.
find the relative elements.

Relative locator methods can take as the argument for the point of origin, either a previously located element reference, or another locator. In these examples we’ll be using locators only, but you could swap the locator in the final method with an element object and it will work the same.

Let us consider the below example for understanding the relative locators.

![Relative Locators](https://ipic.ilovestudy.top/blog/20230514/relative_locators.png)

#### Available relative locators

##### Above

If the email text field element is not easily identifiable for some reason, but the password text field element is, we can locate the text field element using the fact that it is an “input” element “above” the password element.

```java
By emailLocator = RelativeLocator.with(By.tagName("input")).above(By.id("password"));
```

##### Below

If the password text field element is not easily identifiable for some reason, but the email text field element is, we can locate the text field element using the fact that it is an “input” element “below” the email element.

```java
By passwordLocator = RelativeLocator.with(By.tagName("input")).below(By.id("email"));
```

##### Left of

If the cancel button is not easily identifiable for some reason, but the submit button element is, we can locate the cancel button element using the fact that it is a “button” element to the “left of” the submit element.

```java
By cancelLocator = RelativeLocator.with(By.tagName("button")).toLeftOf(By.id("submit"));
```

##### Right of

If the submit button is not easily identifiable for some reason, but the cancel button element is, we can locate the submit button element using the fact that it is a “button” element “to the right of” the cancel element.

```java
By submitLocator = RelativeLocator.with(By.tagName("button")).toRightOf(By.id("cancel"));
```

##### Near

If the relative positioning is not obvious, or it varies based on window size, you can use the near method to identify an element that is at most `50px` away from the provided locator. One great use case for this is to work with a form element that doesn’t have an easily constructed locator, but its associated [input label element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/label) does.

```java
By emailLocator = RelativeLocator.with(By.tagName("input")).near(By.id("lbl-email"));
```

#### Chaining relative locators

You can also chain locators if needed. Sometimes the element is most easily identified as being both above/below one element and right/left of another.

```java
By submitLocator = RelativeLocator.with(By.tagName("button")).below(By.id("email")).toRightOf(By.id("cancel"));
```

