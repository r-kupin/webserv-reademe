Minimalist re-implementation of nginx web server.
# HowTo
**WARNING** 
I am not using `Makefile` in development process, so the **lists of source files might be outdated**, and therefore - project might not compile. However - it is quite easy to get up-to-date source lists:
- Enter repository root
- *SRCS*: all project sources `find src/ -name \*.cpp -print `
- *LIB_SRCS*: *SRCS*, but without `src/main.cpp`
- *TEST_SRCS*: `find test/tests/ -name \*.cpp -print  `
## Run
1. Compile with `make`
2. Launch as `webserv [ path_to_config ]`
3. Connect to the servers on ports defined in the config with any HTTP network-accessing app
## Test
1. Prepare test library 
    ```shell
	git clone git@github.com:google/googletest.git test/lib
	mkdir test/lib/build && cd test/lib/build 
	cmake ..
	```
2. Get back to project's root
3. Run `make test`
# Features
## Done
- Your program has to take a configuration file as argument, or use a default path.
- Choose the [port](#listen) and [host](#server_name) of each server.
- Setup default [error_pages](#error_page).
- Setup routes with one or multiple of the following rules/configuration [return](#return)
	- Define a list of accepted HTTP methods for the route.
	- Define a HTTP redirection.
	- Define a directory or a file from where the file should be searched
	- Make the route able to accept uploaded files and configure where they should be saved
- Set a default file to answer if the request is a directory ([index](#index)).
- Make it work with POST and GET methods.
- Your server must be compatible with the web browser of your choice
- Your HTTP response status codes must be accurate.
- You server must have default error pages if none are provided.
- You must be able to serve a fully static website.
- Limit client body size.
- Clients must be able to upload files
- upload_store implement
- default client_max_body_size 1Mb
## ToDo
- Setup the server_names or not.
- Turn on or off directory listing. (?)
- Your server must be able to listen to multiple ports 
	- multiple domains
	- multiple simultaneous requests to the same server
- A request to your server should never hang forever.
- The first server for a host:port will be the default for this host:port (that means it will answer to all the requests that don’t belong to an other server).
- Execute CGI based on certain file extension (for example .php).
# Config
Like `nginx.conf` but with less functional supported. This project follows philosophy of forward compatibility - meaning that all valid configs for WebServ will be also valid for NGINX, and will work in exact same way.
Feel free to consult the test configs provided in `test/test_resources`. 
## Config structure
Config consists of **contexts** and **directives**.
### Contexts
Context  is a block defined in a following way:
``` nginx
context_name [ ARG ] {
	...
}
```
For now, webserv supports the following contexts:
#### Server
The main context of the instance of the HTTP server. At least one should be defined in the config. 
Server context can't be empty - it should contain mandatory server-level directives: 
- *[server_name](#server_name)* (unique)
- *[listen](#listen)* (unique)

Server also can predefine root location with optional directives:
- *[root](#root)* (unique)
- *[index](#index)*
- *[error_page](#error_page)*
- *[client_max_body_size](#client_max_body_size)* (unique)
- *[upload_store](#upload_store)* (unique)

Inside server context multiple **location** sub-contexts can be defined, to handle specific requests.
```nginx
server {
	listen 4281;  
    server_name localhost;  
    root /var/www;

	location / { ... }
}
```
#### Location
Location sets configuration depending on a request URI. 
Locations can be defined inside of the server or parent location context (nested locations). Server matches the request URI against all defined location, and then assigns handling to the location with the closest matching *address*.
Location context should be defined with a single argument, which is *address*.
```nginx
location address {
	...
}
```
The *address* - is the absolute path from the **root** location, there are no relative paths. It means, that if location *"loc_n"* should be placed inside location *"/loc_1"* - the address should be defined as follows: *"/loc_1/loc_n"*, regardless of whether the super-context is *location /loc_1*, *location /* or *server*:
```nginx
# OK
location / {
	location /loc_1 {
		location /loc_1/loc_2 {
			...
		}
	}
}

location /loc_1/loc_3 {
	....
}
```
But it can't be defined in any context, apart of the mentioned above:
```nginx
# NOT OK
location / {
	location /loc_1 {
		location /loc_3 {
			...
		}
	}
	# "/loc_3" should be defined here
}
# or here
```
Redefinition of locations is possible:
```nginx
# OK
location / {
	location / {
		# This will be the actual version of "/"
		location /loc_1 {
			...
		}
	}
	location /loc_1 {
		...
	}
}

location /loc_1 {
	# And this will be the actual version of "/loc_1"
	... 
}
```
 However one super-location/server can't contain multiple sub-locations with the same addresses:
 ```nginx
# NOT OK
location / {
	
	location /loc_1 {
		...
	}
	
	location /loc_1 {
		...
	}
}
```
Locations also can be mentioned, but not defined explicitly:
```nginx
# OK
server {
	listen 4281;  
    server_name localhost;  
    root /var/www;
	
	location /loc_1/loc_2 {
		# definition of "/loc_1/loc_2"
		...
	}
	# no explicit definition of "/loc_1"
}
```
In this project, such locations referred as **ghost** locations. In the example above, request to `localhost:4281/loc_1/` will lead to `403 Forbidden` server response.
Locations can be empty, or contain following directives:
- *[root](#root)* (unique)
- *[client_max_body_size](#client_max_body_size)* (unique)
- *[upload_store](#upload_store)* (unique)
- *[index](#index)*
- *[return](#return)* (unique)
- *[error_page](#error_page)*

Locations can also contain sub-contexts:
- *[limit_except](#limit_except)* (unique)
- nested *location*
#### Limit_except
Limits access to location. Defined only inside a location with one or more *HTTP* methods:
```nginx
limit_except METHOD {
	...
}
```
The `METHOD` parameter can be one of the following: `GET`, `POST`,  or  `DELETE`, as subject requires. The *limit_except* is a first thing being checked upon the access to the *location*. If request method is not allowed - server immediately responds with *403 Forbidden*.
Limit_except can't be empty, and should contain following directives:
- *deny*
- *allow*

Depending on intention of prohibiting or allowing access. 
Limit_except can't have any sub-contexts.
### Directives
Directive is a single-line instruction defined in a following way:
```nginx
directive [ ARG1 ] [ ARG... ];
```
#### Server-level directives
##### listen
Has only one *arg* which sets the port, used by the server for requests listening.
##### server_name
Should define server's host name, but only works for *localhost* right now
#### Location-level directives
##### root
Can have only one arg, which is a path for a location, or server's root directory. For example:
```nginx
server {
	...
    root /var/www; # absolute path
	
	location /loc_0 {}
	
	location /loc_1 { 
		root resources; # relative path
		location /loc_1/loc_2 {} 
	}
}
```
In this case:
- URI with address `/text.txt` would make server look for `text.txt` in `/var/www/`.
- URI `/loc_0/text.txt` will be handled in `/var/www/loc_0/`, because nginx appends location address to parent location's root, if it isn't overridden.
- URI `/loc_1/text.txt` will be handled by path, constructed as `path to executable` + `resources` + `/loc_1`
- URI `/loc_1/loc_2/text.txt` will be handled by path, constructed as `parrent's root` + `/loc_2`
##### client_max_body_size
Should have only one arg, which is a number in bytes.
Sets bounds for request's body size. Works in the following way: while reading client's body, server keeps track of it's size. If `client_max_body_size` is defined, and client's body exceeds it - server abandon's further request processing and returns error **413**. If not specified - default value of 1Mb is being applied.
##### upload_store
In this project, it's behavior is slightly simplified, because this directive is not a part of vanilla nginx, but from a third-party module, more info [here](#Uploads).
 **Works only for requests done with CURL**
Should have only one arg, which is path to the uploads directory.
Set's path to uploads directory. When location containing this directive handles POST request, it creates a file in specified directory, and writes request's body to it. The name of the file being created is it's number: first is `1`, second is `2`, etc. If File already exists or it's not possible to create it - server returns 503
##### index
May have multiple args that define files that will be used as an index - meaning - shown when location get's accessed by address, following with `/`. Files are checked in the specified order - left to right. The last element of the list can be a file with an absolute path - meaning, path not from the current location's root - but from **root**-location's root.
```nginx
index index_X.html index_1.html index_2.html;
```
Indexes are checked in the following order: 
###### defined in current location
1. Return first index found
2. Return *403* if none of specified files exists & is accessible
###### not defined in current location, but in parent location
Check parent location in the same way, except parent *index filenames* specified in *parent* are expected to be located at the *current location's root*:
```nginx
server {  
		...
        root www;
        index index_X.html index_1.html index_2.html;

        location /loc_1 {
		    # Request /loc_1/
	        # Checks www/loc_1/index_X.html first, /loc_4/index.html - then
	        # Returns 403 if both are not accessible
            index index_X.html /loc_4/index.html;
        }  
  
        location /loc_2 {
	        # Request /loc_2/
	        # Checks www/loc_2/index_X.html, www/loc_2/index_1.html then
	        # www/loc_2/index_2.html
	        # Returns 403 if all are not accessible
        }
}
```
###### no definition up to the root
Default index `index.html` is being checked 
```nginx
server {  
		...
        root www; # webserv
        # no index definition
  
        location /loc_1 {
	        index index_X.html /loc_4/index.html;
		    # Request /loc_1/
	        # Checks www/loc_1/index_X.html first, /loc_4/index.html - then
	        # Returns 403 if both are not accessible
        }  
  
        location /loc_2 {
	        # Request /loc_2/
	        # Checks www/loc_2/index.html
	        # Returns 403 if it is not accessible
        }
}
```
##### return
Directive, responsible for redirection. Stops processing request and returns the specified code to a client. Should have one or two args.
```nginx
location /redirect_no_code {  
    return /target_location;  
}

location /redirect_internal {  
    return 302 /target_location;  
}  
  
location /redirect_external {  
    return 302 http://example.com;  
}  
  
location /target_location {  
    return 200 "Welcome to the target location!";  
}
```
- If return has 1 argument, it is should be a `return code` or `address`.  
- If return has 2 arguments, it is should be a `return code` and `address` or `custom message` - depending on the `return code` value.
- There can't be more than 2 args, and `code` can't be the second arg. 
- The redirect, if only address specified, is done with *302* code
##### error_page
Similar to *index* - it is possible to define custom error pages for each location.
Error_page directive expects one or more `error code`(s) followed by a `filename` of the error page, that should be sent to the client in case if one of the specified errors will happen.
In case, if error page is not defined, or defined file doesn't exist - webserv would *auto generate default error page automatically*.
Example:
```nginx
error_page 403 404 /error.html;
```
# How it actually works?
## Init
### Arg check
In order to work, server needs a config, which should be passed as a one optional argument. If such argument is present, server will try to create a `Config` object which is intended to store `Node`s, each one dedicated to a particular parameter.
In case if provided address doesn't exist, *isn't readable*, if config made with mistakes or there were no arguments at all - server will try to load a default config by the address `resources/nginx.conf`, performing the same checks as for the custom one.
## Setting up Config
### Config
Main class, storing configurations for all servers is `Config`. All its methods are dedicated to parsing config file to list of `ServerConfiguration` classes, each one storing a configuration for each particular server.
### ServerConfiguration
Particular config, the backbone of each server. Contains server-level data, such as *server_name*, *port* and the root of the tree of `Locations`.
`ServerConfiguration`'s functionality is narrowed to function
```c++
LocConstSearchResult    FindConstLocation(const std::string &address) const;
```
that searches the locations tree for a requested location, and returns a `LocSearchResult`, that contains iterator to the closest found location, as well as some additional info.
### Location
Stores data about all [locations](#location) mentioned in config:
```c++
	std::set<ErrPage>       error_pages_;  
    l_loc                   sublocations_;  
//-------------------index related  
    bool                    has_own_index_defined_;  
    bool                    index_defined_in_parent_;  
    l_str                   own_index_;  
  
    Limit                   limit_except_;  
//-------------------redirect related  
    int                     return_code_;  
    std::string             return_internal_address_;  
    std::string             return_external_address_;  
    std::string             return_custom_message_;  
  
    std::string             root_;  
    std::string             full_address_; // address from the root path
    std::string             address_; // particular location's address
    std::string             body_file_; // address of file being sent to client
    l_loc_it                parent_; // root location's "parent" points on itself
    bool                    ghost_;
```
## Setting up servers
## [Request](https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages#http_requests) handling
HTTP request is a message sent by a client to a server:

![request](https://github.com/r-kupin/webserv/blob/main/notes/request.jpg)

Right upon receival of the connection from client, server reads the contents of client request to the `ClientRequest` class.
```c++
//---Request line
Methods                             method_;
//-----------URL
std::string                         addr_;  
std::string                         addr_last_step_;  
bool                                index_request_;  
m_str_str                           params_;
std::string                         fragment_;  
//---Headers & Body
m_str_str                           headers_;
std::string                         body_;  
```
### Request line
-  Method(`method_`): the HTTP method or verb specifies the type of request being made. WebServ is supposed to handle GET, POST and DELETE methods
### [URL](https://developer.mozilla.org/en-US/docs/Learn/Common_questions/Web_mechanics/What_is_a_URL)
- Path (`addr_`): An absolute path, optionally followed by a `'?'` and query string.
-  Last Step in Address (`addr_last_step_`): The contents of the address after the last `/` in URI
- Index Request (`index_request_`): flag indicating whether the request is for the default index resource. WebServ, automatically serves a default file (e.g., index.html) when the path points to a directory meaning if address ends with `/`.
- Fragment (`fragment_`): the fragment identifier, often used in conjunction with anchors in HTML documents. It points to a specific section within the requested resource.
- Parameters (`params_`): additional parameters sent with the request. In the URL, these are typically query parameters (e.g., `?key1=value1&key2=value2`).
### Headers (`headers_`)
HTTP headers provide additional information about the request, such as the type of client making the request, the preferred response format, authentication information, etc.
### Body (`body_`)
The body of the HTTP request, which contains additional data sent to the server. This is particularly relevant for POST requests or other methods where data is sent in the request body. In case of file upload the body will contain file contents.
## [Response](https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages#http_responses) creating
Right after the creation of the `ClientRequest` server starts generating response, which involves 2 steps: creating a synthetic location and creating a `ServerResponse` class
### Creating a synthetic location
Depending on compliance between what was requested and what is being found creates a synthetic location - a copy of the location that was found in [ServerConfig](#ServerConfiguration), but with altered return code, and  redirect-related fields, or with a body file set.
In order to determine what should be returned, server performs some checks:
#### Request's body check
If request contains body, it's size will be counted while reading from socket. If size would exceed limit - error code *413* will be returned.
#### Access permission
Server checks whether access to requested location is prohibited with [`limit_except`](#Limit_except)
#### Redirection
If access is allowed, server then checks defined [redirection](#return), and bounces client with internal or external redirect, or returns specified code and custom message.
#### Server-side handling
##### Upload request
If found location contains [upload_store](#upload_store)  - all requests to it will be treated as uploads. They have to:
- Be POST
- Have headers set:
	- `User-Agent`: only **curl** supported
	- `Content-Type`: should have a `boundary` delimiter - a unique string that separates the individual parts of the message. `boundary` parameter is used to delineate the boundaries between different parts of the message body. 
	- `Content-Length`: corresponds to the size of the request body, which is bigger then the file itself, because it contains some metadata such as filename.
- Have a body of the certain structure:
	1. Start with delimiter preceded by `\r\n--`
	2. Contain some file metadata (optional) followed by `\r\n\r\n` (mandatory)
	3. Contain actual file contents
	4. End up with delimiter preceded by `\r\n--`
If amount of bytes processed corresponds with the value of `Content-Length` and the last thing received was delimiter - request is correct. Otherwise - [400  Bad Request](#400  Bad Request) will be returned.
Before the start of the upload process, server also checks the file being created to store this upload:
- Check that the value of `upload_store` points indeed to the directory where we are supposed to create files
- Check that file intended to store current upload doesn't already exist
- Check that server has permissions to create the file
- Check that server's storage has enough free space to store file
If any of those fails - [503 Service Unavailable](#503)  will be returned
##### CGI request
##### Static request
If found location doesn't contain any directives specifying that request should be uploaded or handled by CGI, server proceeds with checking for the existence of the requested resource.
There are 2 types of static requests - for file and for index. If `path` part of the URL ends with `/` - this is an index request, otherwise - file request. That requests are handled differently.
###### Synthetic location for file request
If `path` part of the URL has something after the last `/` symbol, it is assumed that it is a name of the file, that should be located in the root directory of the location, that preceded the filename. Depending on the result of the file system check server finishes response location:
```c++
if (fs_status == NOTHING) {  
    std::cout << "open() \"" + address + "\" failed" << std::endl;  
    synth.return_code_ = 404;  
} else if (fs_status == DIRECTORY) {  
    // redirect to index request  
    synth.return_code_ = 301;  
    synth.return_internal_address_ = request_address + "/";  
} else {  
    synth.body_file_ = address;  
    synth.return_code_ = 200;  
}
```
###### Synthetic location for index request
At this point, server determines which file should be returned. Server checks index files defined in found location or in parenting ones, or the default `index.html` if nothing were defined at all. More info here:  [index](#index)
Depending on filesystem response status of the directory being requested `fs_status` and of the index file of a particular location - server set's `return_code` and `body_file`:
```c++
if (Utils::CheckFilesystem(index_address) == NOTHING) {  
    // index address not found  
    if (fs_status != DIRECTORY) {  
        // directory, where this index supposed to be doesn't exist  
        std::cout << "\"" + index_address + "\" is not found" << std::endl;  
        synth.return_code_ = 404;  
    } else {  
        // directory exists  but there are no index to return  
        std::cout << "directory index of " + found->root_ +  
                                            "/ is forbidden" << std::endl;  
        synth.return_code_ = 403;  
    }  
} else {  
    // index file found  
    synth.return_code_ = 200;  
    synth.body_file_ = index_address;  
}
```
### Creating `ServerResponse` class
Just as in case with `ClientRequest` class, `ServerResponse` is intended to contain data, corresponding to different parts of server's response message

![response](https://github.com/r-kupin/webserv/blob/main/notes/response.jpg)

Server creates response in a following way:
1. Composes the top part of the HTTP response, including the status line.
2. Adds standard headers like `Server` and `Date`.
3. Determines the content of the response body based on the `Location` object:
	1. If a custom message is provided, it is used.
	2. If it's an error code:
		1. Checks if a custom error page is defined for the given error code in the `Location` object.
		2. If a custom error page is defined, retrieves and sets it as the response body.
		3. If not, generates a generic error page.
	3. If it's a redirection code:
		1. Generates a redirection page as the response body.
		2. If an external or internal address is provided, sets the `Location` header accordingly.
	4. If a body file is specified, its content is read.
4. Sets additional headers like `Content-Type`, `Content-Length`, and `Connection`.

#  Additional info
## Server response codes implemented
### OK
#### 100 Continue
In case if request's body is large, client might ask server for a confirmation before sending body. Client does it by including `Expect: 100-continue` header in request.
In this case, server will send short message to let client start body upload.
#### 200  OK
This status code is returned when the server successfully processes the request and provides the requested resource. It signifies that the client's request has been fulfilled without any issues.

### Implicit redirect
#### 301 Moved Permanently
This status code is returned when client requests for a static file, but specified address actually points to a directory. In this case, response is also followed by a header `Location`, which value corresponds to request url, followed by '/'
### Client side errors
#### 400  Bad Request
If the server cannot process the client's request due to malformed syntax or other errors on the client side, it returns this status code. It indicates that there was an error in the client's request.
#### 403  
#### 404  
#### 405  
#### 413  

### Server side errors
#### 500  
#### 501  
#### 503  
#### 505
## Uploads

![one_does_not_simply](https://github.com/r-kupin/webserv/blob/main/notes/one_does_not_simply.jpg)

Typically, no one uses nginx (or any web server) to store files, uploaded by a client on a machine that runs the server. Upload requests are normally being transferred to the  web application's back-end, that decides what to do with the data: save in database, store on the NAS, etc. In summary - server's job is to transfer requests to appropriate back-end and therefore it doesn't have a simple and direct way to handle uploads.
In order to make real nginx store uploaded files, the most intuitive way I found is described below:
1. Download [**nginx-upload-module**](https://www.nginx.com/resources/wiki/modules/upload/) from the official [github page](https://github.com/vkholodkov/nginx-upload-module/tree/master).
2. Add it to installed server following [this guide](https://www.nginx.com/blog/compiling-dynamic-modules-nginx-plus/).
3. Launch server with test configuration.
	```nginx
	server {
	 	# specify server name, port, root
		location /upload {
		    upload_pass   @test;
		    upload_store /path/to/upload_directory 1;
		}
		
		location @test {
		    return 200 "Hello from test";
		}
	}
	```
4. Use web client to upload file. Here's one of the simplest ways: `curl -v -F "file=@file_name" http://server_name:port/upload/ `

As described at module's page:
-  [upload_pass](https://www.nginx.com/resources/wiki/modules/upload/#upload-pass "Permalink to this headline"): specifies location to pass request body to. File fields will be stripped and replaced by fields, containig necessary information to handle uploaded files.
- [upload_store](https://www.nginx.com/resources/wiki/modules/upload/#upload-store "Permalink to this headline"): specifies a directory to which output files will be saved to. The directory could be hashed. In this case all subdirectories should exist before starting NGINX.

Important notes:
1. *upload_store* won't work if *upload_pass* is not specified - server will return *403* if there is no index file in uploads directory, or *405* otherwise. Nothing will be stored.
2. In  *upload_directory* sub-directories 0 1 2 3 4 5 6 7 8 9 should exist and be accessible by user, that is used by nginx. I have tested nginx on Debian machine with nginx installed via `apt install`, and the username used by nginx was *www-data*, however, it might be different on other systems and/or if other ways of installation were used. In my case, working *upload_directory* looked like this:
	```
	❯ ls -l
	total 40
	dr-xrwxr-x 2 my_login www-data 4096 Jan 14 15:36 0
	dr-xrwxr-x 2 my_login www-data 4096 Jan 14 15:37 1
	dr-xrwxr-x 2 my_login www-data 4096 Jan 14 15:37 2
	dr-xrwxr-x 2 my_login www-data 4096 Jan 14 16:14 3
	dr-xrwxr-x 2 my_login www-data 4096 Jan 14 16:14 4
	dr-xrwxr-x 2 my_login www-data 4096 Jan 14 16:43 5
	dr-xrwxr-x 2 my_login www-data 4096 Jan 14 16:43 6
	dr-xrwxr-x 2 my_login www-data 4096 Jan 14 16:47 7
	dr-xrwxr-x 2 my_login www-data 4096 Jan 14 16:47 8
	dr-xrwxr-x 2 my_login www-data 4096 Jan 14 16:51 9
	```
 	Otherwise, nginx would be unable to create files to save the uploads and will return *503*
 3. The only request method supported is *POST*
 4. Works in a following way: 
	 1. for each request that posts to *upload_directory* nginx creates file with name which is a numbers of files saved preceded with zeros, so each file being saved has name of 10 characters.
	 2. files are placed in 10 directories, based on the last digit:
		 1. `0000000001`, `0000000021`, etc.. - in `./1`
		 2. `0000000010`, `0000000210`, etc.. -  in `./0`
    3. So the first request's body would be saved to  `./1/0000000001`, second - `./2/0000000002`, etc..