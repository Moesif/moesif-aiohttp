# Moesif Middleware for AIOHTTP

[![Built For][ico-built-for]][link-built-for]
[![Latest Version][ico-version]][link-package]
[![Language Versions][ico-language]][link-language]
[![Software License][ico-license]][link-license]
[![Source Code][ico-source]][link-source]

AIOHTTP middleware that automatically logs _incoming_ API calls and sends to [Moesif](https://www.moesif.com) for API analytics and monitoring.

[Source Code on GitHub](https://github.com/moesif/moesif-aiohttp)


## How to install

```shell
pip install moesif_aiohttp
```

## Configuration options

For options that use the request and response as input arguments, these use the `aiohttp` web [request](https://docs.aiohttp.org/en/stable/web_reference.html#aiohttp.web.Request) or [response](https://docs.aiohttp.org/en/stable/web_reference.html#aiohttp.web.Response) objects. 

Please note that incase of the streaming api, the response object is [aiohttp_sse.EventSourceResponse](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)

#### __`APPLICATION_ID`__
(__required__), _string_, is obtained via your Moesif Account, this is required.

#### __`SKIP`__
(optional) _(request, response) => boolean_, a function that takes a request and a response, and returns true if you want to skip this particular event.

#### __`IDENTIFY_USER`__
(optional, but highly recommended) _(request, response) => string_, a function that takes a request and a response, and returns a string that is the user id used by your system. While Moesif tries to identify users automatically,
but different frameworks and your implementation might be very different, it would be helpful and much more accurate to provide this function.

#### __`IDENTIFY_COMPANY`__
(optional) _(request, response) => string_, a function that takes a request and a response, and returns a string that is the company id for this event.

#### __`GET_METADATA`__
(optional) _(request, response) => dictionary_, getMetadata is a function that returns an object that allows you to add custom metadata that will be associated with the event. The metadata must be a dictionary that can be converted to JSON. For example, you may want to save a VM instance_id, a trace_id, or a tenant_id with the request.

#### __`GET_SESSION_TOKEN`__
(optional) _(request, response) => string_, a function that takes a request and a response, and returns a string that is the session token for this event. Again, Moesif tries to get the session token automatically, but if you setup is very different from standard, this function will be very help for tying events together, and help you replay the events.

#### __`MASK_EVENT_MODEL`__
(optional) _(EventModel) => EventModel_, a function that takes an EventModel and returns an EventModel with desired data removed. The return value must be a valid EventModel required by Moesif data ingestion API. For details regarding EventModel please see the [Moesif Python API Documentation](https://www.moesif.com/docs/api?python).

#### __`DEBUG`__
(optional) _boolean_, a flag to see debugging messages.

#### __`LOG_BODY`__
(optional) _boolean_, default True, Set to False to remove logging request and response body.

#### __`EVENT_QUEUE_SIZE`__
(optional) __int__, default 1000000, the maximum number of event objects queued in memory pending upload to Moesif.  If the queue is full additional calls to `MoesifMiddleware` will return immediately without logging the event, so this number should be set based on the expected event size and memory capacity

### __`EVENT_WORKER_COUNT`__
(optional) __int__, default 2, the number of worker threads to use for uploading events to Moesif. If you have a large number of events being logged, increasing this number can improve upload performance.

#### __`BATCH_SIZE`__
(optional) __int__, default 100, Maximum batch size when sending events to Moesif when reading from the queue

#### __`EVENT_BATCH_TIMEOUT`__
(optional) __int__, default 2, Maximum time in seconds to wait before sending a batch of events to Moesif when reading from the queue

#### __`AUTHORIZATION_HEADER_NAME`__
(optional) _string_, A request header field name used to identify the User in Moesif. Default value is `authorization`. Also, supports a comma separated string. We will check headers in order like `"X-Api-Key,Authorization"`.

#### __`AUTHORIZATION_USER_ID_FIELD`__
(optional) _string_, A field name used to parse the User from authorization header in Moesif. Default value is `sub`.

#### __`BASE_URI`__
(optional) _string_, A local proxy hostname when sending traffic via secure proxy. Please set this field when using secure proxy. For more details, refer [secure proxy documentation.](https://www.moesif.com/docs/platform/secure-proxy/#2-configure-moesif-sdk)

### Example:

```python
def identify_user(request, response):
    # Your custom code that returns a user id string
    return "12345"

def identify_company(request, response):
    # Your custom code that returns a company id string
    return "67890"

def should_skip(request, response):
    # Your custom code that returns true to skip logging
    return "health/probe" in request.url

def get_token(request, response):
    # If you don't want to use the standard WSGI session token,
    # add your custom code that returns a string for session/API token
    return "XXXXXXXXXXXXXX"

def mask_event(eventmodel):
    # Your custom code to change or remove any sensitive fields
    if 'password' in eventmodel.response.body:
        eventmodel.response.body['password'] = None
    return eventmodel

def get_metadata(app, environ):
    return {
        'datacenter': 'westus',
        'deployment_version': 'v1.2.3',
    }

moesif_settings = {
    'APPLICATION_ID': 'Your Moesif Application Id',
    'DEBUG': False,
    'LOG_BODY': True,
    'IDENTIFY_USER': identify_user,
    'IDENTIFY_COMPANY': identify_company,
    'GET_SESSION_TOKEN': get_token,
    'SKIP': should_skip,
    'MASK_EVENT_MODEL': mask_event,
    'GET_METADATA': get_metadata,
    'CAPTURE_OUTGOING_REQUESTS': False
}

app = web.Application(
        middlewares=[MoesifMiddleware(moesif_settings)],
    )
```

## Update User

### Update A Single User
Create or update a user profile in Moesif.
The metadata field can be any customer demographic or other info you want to store.
Only the `user_id` field is required.
For details, visit the [Python API Reference](https://www.moesif.com/docs/api?python#update-a-user).

```python
moesif_settings = {
    'APPLICATION_ID': 'Your Moesif Application Id',
}

# Only user_id is required.
# Campaign object is optional, but useful if you want to track ROI of acquisition channels
# See https://www.moesif.com/docs/api#users for campaign schema
# metadata can be any custom object
user = {
  'user_id': '12345',
  'company_id': '67890', # If set, associate user with a company object
  'campaign': {
    'utm_source': 'google',
    'utm_medium': 'cpc', 
    'utm_campaign': 'adwords',
    'utm_term': 'api+tooling',
    'utm_content': 'landing'
  },
  'metadata': {
    'email': 'john@acmeinc.com',
    'first_name': 'John',
    'last_name': 'Doe',
    'title': 'Software Engineer',
    'sales_info': {
        'stage': 'Customer',
        'lifetime_value': 24000,
        'account_owner': 'mary@contoso.com'
    },
  }
}

update_user = MoesifMiddleware(moesif_settings).update_user(user)
```

### Update Users in Batch
Similar to update_user, but used to update a list of users in one batch. 
Only the `user_id` field is required.
For details, visit the [Python API Reference](https://www.moesif.com/docs/api?python#update-users-in-batch).

```python
moesif_settings = {
    'APPLICATION_ID': 'Your Moesif Application Id',
}

userA = {
  'user_id': '12345',
  'company_id': '67890', # If set, associate user with a company object
  'metadata': {
    'email': 'john@acmeinc.com',
    'first_name': 'John',
    'last_name': 'Doe',
    'title': 'Software Engineer',
    'sales_info': {
        'stage': 'Customer',
        'lifetime_value': 24000,
        'account_owner': 'mary@contoso.com'
    },
  }
}

userB = {
  'user_id': '54321',
  'company_id': '67890', # If set, associate user with a company object
  'metadata': {
    'email': 'mary@acmeinc.com',
    'first_name': 'Mary',
    'last_name': 'Jane',
    'title': 'Software Engineer',
    'sales_info': {
        'stage': 'Customer',
        'lifetime_value': 48000,
        'account_owner': 'mary@contoso.com'
    },
  }
}
update_users = MoesifMiddleware(moesif_settings).update_users_batch([userA, userB])
```

## Update Company

### Update A Single Company
Create or update a company profile in Moesif.
The metadata field can be any company demographic or other info you want to store.
Only the `company_id` field is required.
For details, visit the [Python API Reference](https://www.moesif.com/docs/api?python#update-a-company).

```python
moesif_settings = {
    'APPLICATION_ID': 'Your Moesif Application Id',
}

# Only company_id is required.
# Campaign object is optional, but useful if you want to track ROI of acquisition channels
# See https://www.moesif.com/docs/api#update-a-company for campaign schema
# metadata can be any custom object
company = {
  'company_id': '67890',
  'company_domain': 'acmeinc.com', # If domain is set, Moesif will enrich your profiles with publicly available info 
  'campaign': {
    'utm_source': 'google',
    'utm_medium': 'cpc', 
    'utm_campaign': 'adwords',
    'utm_term': 'api+tooling',
    'utm_content': 'landing'
  },
  'metadata': {
    'org_name': 'Acme, Inc',
    'plan_name': 'Free',
    'deal_stage': 'Lead',
    'mrr': 24000,
    'demographics': {
        'alexa_ranking': 500000,
        'employee_count': 47
    },
  }
}

update_company = MoesifMiddleware(moesif_settings).update_company(company)
```

### Update Companies in Batch
Similar to update_company, but used to update a list of companies in one batch. 
Only the `company_id` field is required.
For details, visit the [Python API Reference](https://www.moesif.com/docs/api?python#update-companies-in-batch).

```python
moesif_settings = {
    'APPLICATION_ID': 'Your Moesif Application Id',
}

companyA = {
  'company_id': '67890',
  'company_domain': 'acmeinc.com', # If domain is set, Moesif will enrich your profiles with publicly available info 
  'metadata': {
    'org_name': 'Acme, Inc',
    'plan_name': 'Free',
    'deal_stage': 'Lead',
    'mrr': 24000,
    'demographics': {
        'alexa_ranking': 500000,
        'employee_count': 47
    },
  }
}

companyB = {
  'company_id': '09876',
  'company_domain': 'contoso.com', # If domain is set, Moesif will enrich your profiles with publicly available info 
  'metadata': {
    'org_name': 'Contoso, Inc',
    'plan_name': 'Free',
    'deal_stage': 'Lead',
    'mrr': 48000,
    'demographics': {
        'alexa_ranking': 500000,
        'employee_count': 53
    },
  }
}

update_companies = MoesifMiddleware(moesif_settings).update_companies_batch([companyA, companyB])
```

## Other integrations

To view more documentation on integration options, please visit __[the Integration Options Documentation](https://www.moesif.com/docs/getting-started/integration-options/).__

[ico-built-for]: https://img.shields.io/badge/built%20for-python%20aiohttp-blue.svg
[ico-version]: https://img.shields.io/pypi/v/moesif-aiohttp.svg
[ico-language]: https://img.shields.io/pypi/pyversions/moesif-aiohttp.svg
[ico-license]: https://img.shields.io/badge/License-Apache%202.0-green.svg
[ico-source]: https://img.shields.io/github/last-commit/moesif/moesif-aiohttp.svg?style=social

[link-built-for]: https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface
[link-package]: https://pypi.python.org/pypi/moesif-aiohttp
[link-language]: https://pypi.python.org/pypi/moesif-aiohttp
[link-license]: https://raw.githubusercontent.com/Moesif/moesif-aiohttp/master/LICENSE
[link-source]: https://github.com/Moesif/moesif-aiohttp
