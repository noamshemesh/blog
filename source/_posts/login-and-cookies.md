title: 'Login and Cookies on Android'
id: 53
categories:
  - Android
date: 2013-08-30 11:44:57
tags:
---

If you ever tried to work with cookies and Android you know that it isn't covered well over the net.

The log-in screen, however, is more covered, there are many templates all over the net, but we couldn't find a template that covers the flow from A to Z, with cookies and result code handling.

That's the motivation. Now for the good part:

<!--more-->Let's start with architecture, we are going to use cookies to save the credentials of the user by the server in his device. Just as websites do. Also, one of the requirements is that the user will log-in only when he has to (think about [eBay ](http://www.ebay.com) purchasing process), so we have to handle the 401 HTTP code from node.js when the user has no cookie or cookie credentials are incorrect, and not just open for him the `LoginActivity` on start.

For the authentication and credentials management parts we forked [angular-client-side-auth](https://github.com/fnakstad/angular-client-side-auth) that pretty covered up all the node.js and Angular.js log-in and registration flow, and we wanted the same for android.

When apache HTTP client was the option, it didn't know how to persist cookies, so we serialized them into a `SharedPreferences` object, it's probably not the best solution as it was buggy as hell and a lot of code to write.

Then [loopj asynchronous http client library](http://loopj.com/android-async-http/) came up, this great library knows how to persist cookies with just 2 lines of code:

``` java
PersistentCookieStore cookieStore = new PersistentCookieStore(this);
httpClient.setCookieStore(cookieStore);
```

Cookie issue faded away.

Besides cookies, there is result code handle, if it's 401 we send the user to the `LoginActivity.` Then, according to implementation, the app can post/get the url again:

``` java
public void onFailure(final Throwable t, final String message) {
	new AsyncTask&lt;Void, Void, Void&gt;() {
		@Override
		protected Void doInBackground(Void... params) {
			return null;
		}

		@Override
		protected void onPostExecute(Void result) {
			if (t instanceof HttpResponseException) {
				int responseCode = ((HttpResponseException) t).getStatusCode();
				if (responseCode == 401) {
					commonPool.handleUnauthorize();
				} else {
					String msg = responseCode + &quot;: &quot; + t.getMessage() + &quot; &quot; + message;
					Log.e(TAG, msg);
					onFailure.doCallback(msg);
				}
			}
		}
	}.execute();
}

```

The `AsyncTask` is because we want to run the `startActivity` on the UI thread.

Here is the code for POST one object and return one object and GET and return one object. `CommonPool` is the `Application` of the project.

``` java ServerUtils

public final class ServerUtils {
	protected static final String TAG = ServerUtils.class.getSimpleName();
	private static AsyncHttpClient httpclient;
	private static final String APPLICATION_JSON = &quot;application/json&quot;;
	private static final String TEXT_PLAIN = &quot;text/plain&quot;;
	private static final String CHARSET = &quot;UTF-8&quot;;

	private static Gson gson = new GsonBuilder().create();

	private ServerUtils() {
	}

	private static void initClient(CommonPool commonPool) {
		if (httpclient != null) {
			return;
		}
		httpclient = new AsyncHttpClient();
		PersistentCookieStore cookieStore = new PersistentCookieStore(commonPool);
		httpclient.setCookieStore(cookieStore);
	}

	private static void onPreExecute(CommonPool commonPool) {
		initClient(commonPool);
		httpclient.setTimeout(Constants.DEFAULT_SERVER_TIMEOUT);
	}

	public static  void get(CommonPool commonPool, String url, final Callback onCallback, final Callback onFailure,
			final Class clazz) {
		onPreExecute(commonPool);
		httpclient.get(url, getOneObjectJson(onCallback, clazz, commonPool, onFailure));
	}

	public static  void post(final CommonPool commonPool, String url, final Object object,
			final Callback onCallback, final Callback onFailure, final Class clazz) {
		try {
			onPreExecute(commonPool);
			httpclient.post(commonPool, url, new StringEntity(gson.toJson(object), CHARSET),
					getContentType(object),
					getOneObjectJson(onCallback, clazz, commonPool, onFailure));
		} catch (UnsupportedEncodingException e) {
			// Can't happen.
			Log.e(TAG, e.getMessage());
		}
	}

	private static String getContentType(final Object object) {
		return object == null || object instanceof String ? TEXT_PLAIN : APPLICATION_JSON;
	}

	public static class MyJsonHttpResponseHandler extends JsonHttpResponseHandler {
		private Callback onFailure;
		private CommonPool commonPool;

		public MyJsonHttpResponseHandler(CommonPool commonPool, Callback onFailure) {
			this.commonPool = commonPool;
			this.onFailure = onFailure;
		}

		@Override
		public void onFailure(Throwable t, JSONArray jsonArray) {
			onFailure(t, jsonArray != null ? jsonArray.toString() : &quot;&quot;);
		}

		@Override
		public void onFailure(Throwable t, JSONObject json) {
			onFailure(t, json != null ? json.toString() : &quot;&quot;);
		}

		@Override
		public void onFailure(final Throwable t, final String message) {
			new AsyncTask&lt;Void, Void, Void&gt;() {
				@Override
				protected Void doInBackground(Void... params) {
					return null;
				}

				@Override
				protected void onPostExecute(Void result) {
					if (t instanceof HttpResponseException) {
						int responseCode = ((HttpResponseException) t).getStatusCode();
						if (responseCode == 401) {
							commonPool.handleUnauthorize();
						} else {
							String msg = responseCode + &quot;: &quot; + t.getMessage() + &quot; &quot; + message;
							Log.e(TAG, msg);
							onFailure.doCallback(msg);
						}
					}
				}
			}.execute();
		}
	}

	private static  JsonHttpResponseHandler getOneObjectJson(final Callback onCallback, final Class clazz,
			CommonPool commonPool, Callback onFailure) {
		return new MyJsonHttpResponseHandler(commonPool, onFailure) {
			@Override
			public void onSuccess(int reponseCode, JSONObject json) {
				onCallback.doCallback(gson.fromJson(json.toString(), clazz));
			}

			@SuppressWarnings(&quot;unchecked&quot;)
			@Override
			public void onSuccess(int responseCode, String response) {
				if (clazz == null || clazz.isAssignableFrom(String.class)) {
					onCallback.doCallback((T) response);
				} else {
					onFailure(new IllegalArgumentException(&quot;clazz&quot;), response);
				}
			}
		};
	}

	public static String toJson(Object obj) {
		return gson.toJson(obj);
	}

	public static  T fromJson(String json, Class clazz) {
		return gson.fromJson(json, clazz);
	}
}
```

A reasonable parameter to these 2 methods could be a boolean that tells if send to login screen if not logged in, or just return null, that's for some calls that will happen before the critical section that require the login (register [GCM](https://developer.android.com/google/gcm/gs.html) key for example).

Although it may ended up with pretty straight forward code, the way to it wasn't, I hope it will help someone out there.

Noam.