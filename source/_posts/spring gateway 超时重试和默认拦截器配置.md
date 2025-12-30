---
title: spring gateway 超时重试和默认拦截器配置
date: 2023-12-20 21:22
tags: java
categories: 
---

<!--more-->

```YAML
spring:
  cloud:
    gateway:
      default-filters:
        - name: Retry
          args:
            retries: 3
```

RetryConfig 中默认的异常处理为IOException.class, TimeoutException.class

```java
public static class RetryConfig implements HasRouteId {

		private String routeId;

		private int retries = 3;

		private List<Series> series = toList(Series.SERVER_ERROR);

		private List<HttpStatus> statuses = new ArrayList<>();

		private List<HttpMethod> methods = toList(HttpMethod.GET);

		private List<Class<? extends Throwable>> exceptions = toList(IOException.class, TimeoutException.class);

		private BackoffConfig backoff;

		public RetryConfig allMethods() {
			return setMethods(HttpMethod.values());
		}

		public void validate() {
			Assert.isTrue(this.retries > 0, "retries must be greater than 0");
			Assert.isTrue(!this.series.isEmpty() || !this.statuses.isEmpty() || !this.exceptions.isEmpty(),
					"series, status and exceptions may not all be empty");
			Assert.notEmpty(this.methods, "methods may not be empty");
			if (this.backoff != null) {
				this.backoff.validate();
			}
		}

		public BackoffConfig getBackoff() {
			return backoff;
		}

		public RetryConfig setBackoff(BackoffConfig backoff) {
			this.backoff = backoff;
			return this;
		}

		public RetryConfig setBackoff(Duration firstBackoff, Duration maxBackoff, int factor,
				boolean basedOnPreviousValue) {
			this.backoff = new BackoffConfig(firstBackoff, maxBackoff, factor, basedOnPreviousValue);
			return this;
		}

		@Override
		public void setRouteId(String routeId) {
			this.routeId = routeId;
		}

		@Override
		public String getRouteId() {
			return this.routeId;
		}

		public int getRetries() {
			return retries;
		}

		public RetryConfig setRetries(int retries) {
			this.retries = retries;
			return this;
		}

		public List<Series> getSeries() {
			return series;
		}

		public RetryConfig setSeries(Series... series) {
			this.series = Arrays.asList(series);
			return this;
		}

		public List<HttpStatus> getStatuses() {
			return statuses;
		}

		public RetryConfig setStatuses(HttpStatus... statuses) {
			this.statuses = Arrays.asList(statuses);
			return this;
		}

		public List<HttpMethod> getMethods() {
			return methods;
		}

		public RetryConfig setMethods(HttpMethod... methods) {
			this.methods = Arrays.asList(methods);
			return this;
		}

		public List<Class<? extends Throwable>> getExceptions() {
			return exceptions;
		}

		public RetryConfig setExceptions(Class<? extends Throwable>... exceptions) {
			this.exceptions = Arrays.asList(exceptions);
			return this;
		}

	}
```