---
layout: post
title: "Using beforeShowDay from BackBean values"
date: 2015-11-12 17:10:07 +0100
comments: true
categories: [java, jsf, primefaces]
---

I have found some info and examples about using `beforeShowDay` attribute to enable/disable days in `p:calendar` component from PrimeFaces, it is easy and strightforward. But, these examples did not retrieve data from back bean, instead it calculate some conditions on JavaScript.

After some reviews on StackOverflow I found a [great entry](https://zenidas.wordpress.com/recipes/primefaces-calendar-customization/) in Zenida's blog, about how to do it. 

However, this approach did not work on Chrome browser, neither Internet Explorer. So I started debugging and fixed some bugs. Now here you are with a complete solution valid for every browser.

<!-- more -->

I have created a github repository https://github.com/malaguna/samples to build empty projects to test or show some behaviour. Inside this repository I have crieated a *beforeShowDay* maven project. You can check it out, it is fully working.

## Special considerations

Here you will find what are the relevant parts that differs with normal usage of prime faces components, and those that bugged me while doing this job.

### Including JavaScript into xhtml

It is not possible to include a JavaScript function inside an xhtml page anyhow. To avoid problems, the best way is to use `<h:outputScript>` and `CDATA` as follows:

```xml
	<h:outputScript>
	 //<![CDATA[
		function processDay(date) {
			//Your code goes here ...
		};
	 //]]>
	</h:outputScript>
```

It is very important to use double backslash before `CDATA` starts and also at the end.

### Calling back bean from JavaScript

If you want to call a back bean method from JavaScript function, you have to use JSTL functions. In order to use JSTL functions inside an xhtml page, you have to include the namespace `xmlns:fn="http://java.sun.com/jsp/jstl/functions"`.

Now you can use it to invoke a back bean method. For example, the following line will call `calendarBean.getSpecialDays();` that returns a Java array of strings. This array is **joined** into a JavaScript string with comma separated elements, that are converted into a JavaScript array of strings. I know it seems a bit confusing.  

```javascript
	var specialDays = new Array(#{fn:join(calendarBean.getSpecialDays(), ',')}); 
```

### Date format

Due to I want to compare dates inside JavaScript, it is necessary to send dates from back bean to JavaScript well formated. Following formatter is used into Java code:

```java
	private SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
```

Finally, inside JavaScript code you must compare dates ignoring time and timezone information, so previosly it is necessary to transform dates as following:

```javascript
	var compare1 = new Date(date.getFullYear(),date.getMonth(),date.getDate());
```

So now you can compare dates by means of `getTime()` function.

## Resulting code

So, here you have relevant xhtml code:

```xml
<html xmlns="http://www.w3.org/1999/xhtml"
	xmlns:h="http://java.sun.com/jsf/html"
	xmlns:f="http://java.sun.com/jsf/core"
	xmlns:p="http://primefaces.org/ui"
	xmlns:fn="http://java.sun.com/jsp/jstl/functions">

	...

	<h:outputScript>
	 //<![CDATA[
		function processDay(date) {
			var specialDays = new Array(#{fn:join(calendarBean.getSpecialDays(), ',')});
			
			for (var i = 0; i < specialDays.length; i++) {
				var sDate = new Date(specialDays[i]);
				
				var compare1 = new Date(date.getFullYear(),date.getMonth(),date.getDate());
				var compare2 = new Date(sDate.getFullYear(),sDate.getMonth(),sDate.getDate());
				
				if(compare1.getTime() == compare2.getTime()) {
					return [false, ''];
				}
			}
	        
			return [true, ''];
		};
	 //]]>
	</h:outputScript>

	...

	<h:form id="form">
		<h:panelGrid columns="2">
			<h:outputLabel value="Calendario :" />
			<p:calendar id="calendar" mode="inline"
				value="#{calendarBean.currentDay}" beforeShowDay="processDay" />
		</h:panelGrid>
	</h:form>

	...
```
and back bean method

```java
	public String[] getSpecialDays() {
		String[] result = new String[3];

		Calendar cal = Calendar.getInstance();

		// yesterday
		cal.add(Calendar.DATE, -1);
		result[0] = String.format("'%s'", sdf.format(cal.getTime()));

		// Today
		cal.add(Calendar.DATE, 1);
		result[1] = String.format("'%s'", sdf.format(cal.getTime()));

		// Tomorrow
		cal.add(Calendar.DATE, 1);
		result[2] = String.format("'%s'", sdf.format(cal.getTime()));

		return result;
	}
```
This method only returns three *special* days, yesterday, today and tomorrow, as you can see, it is quite easy to calculate other days and return it.

Hope you find it useful.
