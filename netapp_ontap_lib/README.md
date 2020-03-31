# NetApp ONTAP
The Python client library is a package you can use when writing scripts to access the
ONTAP REST API. It provides support for several underlying services, including connection
management, asynchronous request processing, and exception handling. By using the Python
client library, you can quickly develop robust code to support the automation of your ONTAP
deployments.

# Getting started

## Software requirements
Before installing the Python client library, you must make sure the following packages are
installed on your system:

* python 3.5 or later
* requests 2.21.0 or later
* marshmallow 3.2.1 or later


## Installing and importing the package
You can install the latest released version of the package from PyPi using the pip utility:

```
python3 -m pip install netapp-ontap
```

Or you can install the package directoy from the sources in this repo in [development
mode](https://pip.pypa.io/en/stable/reference/pip_install/#editable-installs) if you are testing out changes:

```
python3 -m pip install -e .
```

After installing the package, you can import the objects you need into your application:

```python
from netapp_ontap.resources import Volume, Snapshot
```

## Creating an object

You can create an object in several different ways. Here are three examples of
creating an equivalent `netapp_ontap.resources.volume` object.

```python
from netapp_ontap.resources import Volume

# Example 1 - keyword arguments
volume = Volume(name='vol1', svm={'name': 'vs1'}, aggregates=[{'name': 'aggr1'}])

# Example 2 - dict as keyword arguments
data = {
    'name': 'vol1',
    'svm': {'name': 'vs1'},
    'aggregates': [{'name': 'aggr1'}],
}
volume = Volume(**data)

# Example 3 - using the from_dict() method
volume = Volume.from_dict({
    'name': 'vol1',
    'svm': {'name': 'vs1'},
    'aggregates': [{'name': 'aggr1'}],
})
```

## Performing actions on an object

After you create an object, you can perform actions on the object based
on the purpose and design of your application. The example below illustrates
how to create a new volume and then take a snapshot.

Note that when using the library, in all cases you must first establish a
connection to the management LIF of the ONTAP system using the
`netapp_ontap.host_connection.HostConnection` object. In the example below,
the connection is created and then set as the global default.
This means that all objects and the associated actions reuse
this same connection. See *Host connections* for more information.

```python
from netapp_ontap import config
from netapp_ontap.host_connection import HostConnection
from netapp_ontap.resources import Volume, Snapshot

config.CONNECTION = HostConnection('myhost.mycompany.com', 'username', 'password')

volume = Volume(name='vol1', svm={'name': 'vs1'}, aggregates=[{'name': 'aggr1'}])
volume.post()
snapshot = Snapshot.from_dict({
    'name': '%s_snapshot' % volume.name,
    'comment': 'A snapshot of %s' % volume.name,
    'volume': volume.to_dict(),
})
snapshot.post()
```

# Host connections

The `netapp_ontap.host_connection.HostConnection` object allows a client application
to store credentials once and reuse them for each subsequent operation.
You can do this in any of the following ways:

* Use the connection object as a context manager with the **with** keyword.

* Call the function `set_connection()` on a specific resource so the connection is used for
all actions on the resource.

* Set the `netapp_ontap.config.CONNECTION` variable to establish a single connection instance for all
operations within the scope of that block. This allows you to connect to ONTAP once
and use the same connection everywhere, instead of providing credentials every time you make a
request.

Note that you can call `get_connection()` to get the connection used by an object and use it for
subsequent operations.

By default, every operation attempts to verify the SSL certificate for the connection. If a
certificate cannot be verified, the **SSLError** exception is thrown. You can disable this
verification by setting `netapp_ontap.host_connection.HostConnection.verify` to false when creating the
`netapp_ontap.host_connection.HostConnection` instance.

## Custom headers

In some cases, you might want to set and send custom headers with the REST request.
This can be done at the connection level. For a specific connection, you can pass in
the headers you would like to send for each request within the scope of that connection object.
The library provides full access to the request headers so that you can update, add, or delete
headers from the same connection object at any point. If a header is not recognized by ONTAP,
it is ignored.

```python
from netapp_ontap import config, HostConnection
headers = {'my-header1':'my-header-value1', 'my-header2':'my-header-value2'}

#Initialize a connection object with custom headers
config.CONNECTION = HostConnection('myhost.mycompany.com', 'username', 'password', headers=headers)

#Delete a header from a connection object
conn = HostConnection('myhost.mycompany.com', 'username', 'password', headers=headers)
del conn.request_headers['my-header1']

#Add a header to a connection object using the assignment operator
conn = HostConnection('myhost.mycompany.com', 'username', 'password', headers=headers)
conn.request_headers['mynew-header'] = 'mynew-header-value'

#Add headers to a connection object
config.CONNECTION = HostConnection('myhost.mycompany.com', 'username' 'password')
config.CONNECTION.request_headers = headers

#Update an existing header using the assignment operator
config.CONNECTION = HostConnection('myhost.mycompany.com','username','password', headers=headers)
config.CONNECTION.request_headers['my-header1'] = 'my-new-header'
```

# Asynchronous processing and jobs

All POST, PATCH, and DELETE requests that can take more than two seconds to complete are
designed to run asynchronously as non-blocking operations. These operations are executed
as background jobs at the ONTAP cluster. The HTTP response generated by an
asynchronous request always contains a link to the associated job object. By default, an
asynchronous request automatically polls the job using the unique job identifier in the link.
Control is returned to your script when a terminal state is reached (success or failure) or
the configured timeout value expires. However, you can override this behavior by setting the
**poll** value to false when calling the function, causing control to return before the job
completes. Forcing an immediate return can be useful when a job might take a long time to
complete and you want to continute execution of the script.

# Responses

A request always returns a `netapp_ontap.response.NetAppResponse` object which contains the details
of the HTTP response. It contains information such as whether the response is an error
or a job. Refer to `netapp_ontap.response.NetAppResponse` for further information on how
to check the details of the response.

# Exception handling

By default, an exception is returned if a request returns an HTTP status code of 400 or greater.
The exception object, which is of type `netapp_ontap.error.NetAppRestError`,
holds the HTTP response object so that the exception can be handled in the client code.
If you wish not to raise exceptions, you can set `netapp_ontap.config.RAISE_API_ERRORS` to false. In this case,
it is up to the client to check the HTTP response from the `netapp_ontap.response.NetAppResponse`
object and handle any errors. Refer to `netapp_ontap.error.NetAppRestError` for further information.

```python
# Set RAISE_API_ERRORS to False and check the HTTP response.
config.RAISE_API_ERRORS = False
response = Svm.find(name = "nonexistent_vs)
assert "entry doesn't exist" in response.http_response.text
```

# Debugging

While writing your application, it can often be useful to see the raw HTTP request and response
text that the library is sending to and from the server. There are two flags that can be set
to help with this.

## DEBUG flag

The first is the DEBUG flag. This can be set either by setting DEBUG=1 in the environment prior
to executing your application or by setting `netapp_ontap.utils.DEBUG` to 1 inside of your application.
This flag, when set, will cause the library to log the request and response for any failed
API call. This will be logged at DEBUG level (see the section on logging for setting up your
application). Here's an example of setting this value inside of your application:

```python
import logging

from netapp_ontap import HostConnection, NetAppRestError, config, utils
from netapp_ontap.resources import Volume

logging.basicConfig(level=logging.DEBUG)
config.CONNECTION = HostConnection('10.100.200.50', username='admin', password='password', verify=False)

# Set the DEBUG flag to 1
utils.DEBUG = 1

# this API call will fail with a 404
try:
    volume = Volume(uuid="1", name='does_not_exist')
    volume.get()
except NetAppRestError:
    print('We got an expected exception')
```

Here is what the output would look like:

```
$ python test_debug.py
DEBUG:urllib3.util.retry:Converted retries value: 5 -> Retry(total=5, connect=None, read=None, redirect=None, status=None)
DEBUG:urllib3.connectionpool:Starting new HTTPS connection (1): 10.100.200.50:443
DEBUG:urllib3.connectionpool:https://10.100.200.50:443 "GET /api/storage/volumes/1 HTTP/1.1" 404 130
DEBUG:netapp_ontap.utils:
-----------REQUEST-----------
GET https://10.100.200.50:443/api/storage/volumes/1
Accept: */*
User-Agent: python-requests/2.21.0
Connection: keep-alive
Accept-Encoding: gzip, deflate
X-Dot-Client-App: netapp-ontap-python-9.8.0
Authorization: Basic YWRtaW46cGFzc3dvcmQK
None
-----------------------------

-----------RESPONSE-----------
404 Not Found
Date:Tue, 12 Nov 2019 13:00:24 GMT
Server:libzapid-httpd
X-Content-Type-Options: nosniff
Cache-Control: no-cache,no-store,must-revalidate
Content-Length: 130
Content-Type: application/hal+json
Keep-Alive: timeout=5, max=100
Connection:Keep-Alive
{
  "error": {
    "message": "\"1\" is an invalid value for field \"uuid\" (<UUID>)",
    "code": "2",
    "target": "uuid"
  }
}
------------------------------
We got an expected exception
$
```

## LOG_ALL_API_CALLS flag

There is also a LOG_ALL_API_CALLS flag which can be set in the same ways. You can
set it in the environment or during script execution by setting `netapp_ontap.utils.LOG_ALL_API_CALLS`
to 1. This flag will produce the same output as above, but it will log the call no
matter if there was a failure or not. Here's an example of what that would look
like if we got an existing volume:

```python
import logging

from netapp_ontap import HostConnection, config, utils
from netapp_ontap.resources import Volume

logging.basicConfig(level=logging.DEBUG)
config.CONNECTION = HostConnection('10.100.200.50', username='admin', password='password', verify=False)

# Set the LOG_ALL_API_CALLS flag to 1
utils.LOG_ALL_API_CALLS = 1

# this API call will succeed and be logged
volume = list(Volume.get_collection())[0]
```

Here is what the output would look like:

```
$ python test_debug.py
DEBUG:urllib3.util.retry:Converted retries value: 5 -> Retry(total=5, connect=None, read=None, redirect=None, status=None)
DEBUG:urllib3.connectionpool:Starting new HTTPS connection (1): 10.100.200.50:443
DEBUG:urllib3.connectionpool:https://10.100.200.50:443 "GET /api/storage/volumes HTTP/1.1" 200 567
DEBUG:netapp_ontap.utils:
-----------REQUEST-----------
GET https://10.100.200.50:443/api/storage/volumes
User-Agent: python-requests/2.21.0
Connection: keep-alive
Accept: */*
Accept-Encoding: gzip, deflate
X-Dot-Client-App: netapp-ontap-python-9.8.0
Authorization: Basic YWRtaW46cGFzc3dvcmQK
None
-----------------------------

-----------RESPONSE-----------
200 OK
Date:Tue, 12 Nov 2019 13:14:01 GMT
Server:libzapid-httpd
X-Content-Type-Options: nosniff
Cache-Control: no-cache,no-store,must-revalidate
Content-Length: 567
Content-Type: application/hal+json
Keep-Alive: timeout=5, max=100
Connection:Keep-Alive
{
  "records": [
    {
      "uuid": "c68bdca8-d090-11e9-bb29-005056bb7f42",
      "name": "vs0_root",
      "_links": {
        "self": {
          "href": "/api/storage/volumes/c68bdca8-d090-11e9-bb29-005056bb7f42"
        }
      }
    },
    {
      "uuid": "ed3b6ebf-d48e-11e9-bb29-005056bb7f42",
      "name": "vs1_root",
      "_links": {
        "self": {
          "href": "/api/storage/volumes/ed3b6ebf-d48e-11e9-bb29-005056bb7f42"
        }
      }
    }
  ],
  "num_records": 2,
  "_links": {
    "self": {
      "href": "/api/storage/volumes"
    }
  }
}
------------------------------
$
```

# Additional considerations

In most cases, the objects and actions in the library can be mapped directly
to equivalent cURL commands run against the ONTAP REST interface. However, there are a few
exceptions you should be aware of.

## Property names

If a property of a resource is named the same as one of the Python reserved names,
the name is transposed when accessing the member of the resource. For example,
if there is a resource named "Foo" that has a property defined in the API named "class",
the property name would instead be "class_" when using the library. For example:

```python
from netapp_ontap.resources import Foo

foo = Foo()
foo.class_ = "high"
```

# Documentation
To view the full documentation, visit https://library.netapp.com/ecmdocs/ECMLP2858435/html/index.html
or to view an offline copy, see the `<python_environment>/lib/<python_version>/site_packages/netapp_ontap/docs`
Documentation of ONTAP's REST APIs and other helpful resources can be found at https://devnet.netapp.com/restapi.

# Compatibility

The version assigned to the library consists of the major ONTAP release it is generated
from and a minor version for the library within that release. The minor version allows the
library to be updated within the same ONTAP release. For example, valid versions for
the library associated with ONTAP 9.6 include 9.6.0, 9.6.1, and so on.

Client libraries that have the same major version as ONTAP are completely compatible.
For example, the libraries netapp-ontap-9.6.1 and netapp-ontap-9.6.4 are fully
compatible with both ONTAP 9.6 and ONTAP 9.6P1.

A client library with a major version that does not match the ONTAP release can still be
used, however it will not be fully compatible with the REST API. For example, the library
netapp-ontap-9.6.4 is only partially compatible with ONTAP 9.7. In these cases, the
library may encounter unknown fields or APIs. When this occurs, the library will ignore
unknown fields, return an error, or raise a runtime exception.

# Copyright, trademarks, and feedback
## Copyright information
Copyright &copy; 2019 NetApp, Inc. All Rights Reserved. Printed in the U.S.

No part of this document covered by copyright may be reproduced in any form or by any means&#8208;graphic,
electronic, or mechanical, including photocopying, recording, taping, or storage in an electronic
retrieval system&#8208;without prior written permission of the copyright owner.

Software derived from copyrighted NetApp material is subject to the following license
and disclaimer:

THIS SOFTWARE IS PROVIDED BY NETAPP "AS IS" AND WITHOUT ANY EXPRESS OR IMPLIED WARRANTIES,
INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A
PARTICULAR PURPOSE, WHICH ARE HEREBY DISCLAIMED. IN NO EVENT SHALL NETAPP BE LIABLE FOR ANY
DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)ARISING IN
ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

NetApp reserves the right to change any products described herein at any time, and without notice.
NetApp assumes no responsibility or liability arising from the use of products described herein,
except as expressly agreed to in writing by NetApp. The use or purchase of this product does not
convey a license under any patent rights, trademark rights, or any other intellectual property
rights of NetApp. The product described in this manual may be protected by one or more U.S.
patents, foreign patents, or pending applications.

RESTRICTED RIGHTS LEGEND: Use, duplication,or disclosure by the government is subject to
restrictions as set forth in subparagraph (c)(1)(ii) of the Rights in Technical Data and
Computer Software clause at DFARS 252.277-7103 (October 1988) and FAR 52-227-19 (June 1987).

## Trademark information
NETAPP, the NETAPP logo, and the marks listed on the NetApp Trademarks page are trademarks of
NetApp, Inc. Other company and product names may be trademarks of their respective owners.
http://www.netapp.com/us/legal/netapptmlist.aspx

## Feedback
If you have questions about the library, suggestions, or find a bug, you may contact
by email.

<ng-ontap-rest-python-lib@netapp.com>

You can help us to improve the quality of our documentation by sending us your feedback.
If you have suggestions for improving this document, send us your comments by email.

<doccomments@netapp.com>

To help us direct your comments to the correct division, include in the subject line
the product name, version, and operating system.

If you want to be notified automatically when production-level documentation is released
or important changes are made to existing production-level documents,
follow Twitter account @NetAppDoc.

You can also contact us in the following ways:

NetApp, Inc., 1395 Crossman Ave, Sunnyvale, CA 94089 U.S.

Telephone: +1 (408) 822-6000

Fax: +1 (408) 822-4501

Support telephone: +1 (888) 463-8277