# Varnish QuickJS integration

This repository contains a QuickJS example program combined with a static site builder that forms a simple website. It is built as a Linux ELF static executable and can be uploaded to a running Varnish instance. Once it has been uploaded, it will become the active program for the tenant it has been uploaded to. If you make requests to Varnish directed at the given tenant it will run the current program that has been uploaded and produce content based on the logic of the program. In other words, if you upload a program that produces `Hello World!`, then that's what you will get when you make a request to the given tenant after uploading that program.

You can change the active tenant you are uploading to by modifying `scripts/upload.sh` and change the Host header to any of the tenants. Available tenants:

`Host: jpizza.com`

`Host: vpizza.com`

`Host: wpizza.com`

`Host: xpizza.com`

`Host: ypizza.com`

`Host: zpizza.com`

## Setup

Run `./scripts/setup.sh`.

## Build and upload program

Run `./build_website.sh`

## View your changes

View jpizza.com/j:

http://sandbox.varnish-software.com:8080/j

Example POST to jpizza.com:
```sh
$ curl -D - -d "@text.json" -X POST http://sandbox.varnish-software.com:8080/example -H "Host: jpizza.com" -w @format.txt
```

cURL demonstration:
```sh
curl -X POST http://sandbox.varnish-software.com:8080/example -H "Host: jpizza.com"
```

### Shortcuts for tenants

http://sandbox.varnish-software.com:8080/j --> `Host: jpizza.com`

http://sandbox.varnish-software.com:8080/v --> `Host: vpizza.com`

http://sandbox.varnish-software.com:8080/w --> `Host: wpizza.com`

http://sandbox.varnish-software.com:8080/x --> `Host: xpizza.com`

http://sandbox.varnish-software.com:8080/y --> `Host: ypizza.com`

http://sandbox.varnish-software.com:8080/z --> `Host: zpizza.com`


## JavaScript How To

Information about QuickJS can be found here: https://bellard.org/quickjs/quickjs.html

This program is using [src/my.js](src/my.js) as the source of JavaScript. The `os` and `std` modules are also available. There is a small API towards Varnish that is documented in the header of the source file.

The global scope of the my.js file will be run on initialization. From there you can read the program arguments, read files etc. The third argument to `scriptArgs`, namely `scriptArgs[3]` contains the path of the only file a tenant is allowed to write to. That file can be used to implement basic persistence, but is not going survive kernel panics and power loss.

Once an HTTP request comes in to Varnish, it will go to the given tenant, find the currently active program, and then execute a function in that program. In the QuickJS program the main entry is `my_backend`. The URL is provided as single argument. If anyone makes a POST request, the `my_post_backend` function is called instead, with an extra data argument containing the whole POST body.

## API documentation

> varnish.response(code, content_type, content)

When called from `my_backend` or `my_post_backend` it takes the arguments and immediately produces that response. The VM program stops running and all changes to the VM are discarded.

> varnish.sendfile(path)

If the file exists in the `www` folder, the static site builder would have embedded it automatically, and it can be referred to by the sendfile function. If the file doesn't exist, a 404 is produced instead, making it safe to use this as a fallback for unhandled paths.

> varnish.fetch(url)

Fetch the contents at URL and return an object that contains the status code, content type and contents:
```js
var resp = varnish.fetch("example.com");
varnish.response(resp.status, resp.type, resp.content);
```
Here, resp contains .status, .type and .content. Throws on any error.

> varnish.vmcommit()

Take the current changes done in the VM and make those changes available to every future request. In other words, if you change a global variable and call vmcommit(), that change is now visible to everyone who makes a request after the call. This call is primarily used to learn and predict incoming requests, and is a unique and powerful performance aid. For serialized changes, you can call vmcommit() from a storage VM instead.

> varnish.storage("function_name")

> varnish.storage("function_name", data)

Enter storage by locking it, calling the given function by name, passing data optionally, and then unlocking the storage afterwards. Storage is fundamentally accessed serially, making it possible to have coherent statefulness in programs and do things like counting.

The storage function you enter is a normal JS function, and can optionally be returned from with a string. That string is then passed back to the calling VM.

```js
function my_storage(data) {
	return data;
}

var result = varnish.storage("my_storage", "123");
result == "123"
```

Storage VMs are not ephemeral like request VMs. They can also store their state through updates and even Varnish restarts given that the program writer implements this through a state file. Not covered here.

## Layout

- build_website.sh
	- Builds the full VM program and uploads it
- static_builder.py
	- A helper program that generates an embedded static site
- [src/](src/)
	- my.js: The main JavaScript source file
	- main.c: Initialization and setup
	- http.c: cURL fetcher
	- api.h: API towards KVM
- [quickjs/](quickjs/)
	- QuickJS repository, unmodified
- [scripts/](scripts/)
	- setup.sh: Run `./scripts/setup.sh` after clone
	- upload.sh: Contains some not-secret information on how to upload new programs to a Varnish test instance
- [www/](www/)
	- index.html: Main HTML of the static site portion

The www folder is recursively embedded into the program.
