# REST API: Creating a new selection

In order to create a new selection using the REST API, you need to send an HTTP POST request to the following URL. The selection will then be created, nested underneath the database.

`https://api.copernica.com/v1/database/$id/views?access_token=xxxx`

In this, $id should be replaced by the numerical identifier, the ID, of the database you want to add a selection to. The name of the selection is not in the URL, as it needs to be added to the message body of the HTTP request.

## Available parameters
The following variables can be added to the body of the HTTP POST call:

- **name**: The name of the selection that is to be created. (mandatory)
- **description**: an optional description of the new selection.

## PHP example
The following example demonstrates how to use the API method:

```PHP
// dependencies
require_once('copernica_rest_api.php');

// change this into your access token
$api = new CopernicaRestApi("your-access-token");

// data to pass to the call
$data = array(
    'name'      =>  'my-selection',
);

// do the call
$api->post("database/1234/views", $data);
```
This example uses the [CopernicaRestAPi class](rest-php).

## More information
- [Overview of all API calls](rest-api)
- [Requesting selections of a database](rest-get-database-views)
- [Requesting selection rules](rest-get-view-rules)
- [Creating selection rules](rest-post-view-rules)