---
title: Spel表达demo
date: 2023-03-08 21:59
tags: java
categories: 
---

<!--more-->

```java

package com.example.demo;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.expression.MapAccessor;
import org.springframework.expression.Expression;
import org.springframework.expression.common.TemplateParserContext;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;

import java.util.Collections;
import java.util.Map;

@SpringBootTest
class DemoApplicationTests {

	@Test
	void test1() {

		TemplateParserContext templateParserContext = new TemplateParserContext("${","}");
		StandardEvaluationContext context = new StandardEvaluationContext();
		context.addPropertyAccessor(new MapAccessor());
		context.setRootObject(Map.of(
				"map", Collections.singletonMap("a","b"),
				"test","this is a test"
		));
		SpelExpressionParser parser = new SpelExpressionParser();
		Expression expression = parser.parseExpression("map.a : ${map.a}",templateParserContext);
		Object value = expression.getValue(context);
		System.out.println(value);
	}


	@Test
	void test2(){
		StandardEvaluationContext context = new StandardEvaluationContext();
		context.setVariable("test","this is a test");
		SpelExpressionParser parser = new SpelExpressionParser();
		System.out.println(parser.parseExpression("#test").getValue(context));
	}
}

```