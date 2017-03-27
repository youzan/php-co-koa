## Koa::Request

Request 组件是对swoole_http_request 的简单封装

```php
<?php

class Request
{
    /** @var Application */
    public $app;
    /** @var \swoole_http_request */
    public $req;
    /** @var \swoole_http_response */
    public $res;
    /** @var Context */
    public $ctx;
    /** @var Response */
    public $response;
    /** @var string */
    public $originalUrl;
    /** @var string */
    public $ip;

    public function __construct(Application $app, Context $ctx,
                                \swoole_http_request $req, \swoole_http_response $res)
    {
        $this->app = $app;
        $this->ctx = $ctx;
        $this->req = $req;
        $this->res = $res;
    }

    public function __get($name)
    {
        switch ($name) {
            case "rawcontent":
                return $this->req->rawContent();
            case "post":
                return isset($this->req->post) ? $this->req->post : [];
            case "get":
                return isset($this->req->get) ? $this->req->get : [];
            case "cookie":
            case "cookies":
                return isset($this->req->cookie) ? $this->req->cookie : [];
            case "request":
                return isset($this->req->request) ? $this->req->request : [];
            case "header":
            case "headers":
                return isset($this->req->header) ? $this->req->header : [];
            case "files":
                return isset($this->req->files) ? $this->req->files : [];
            case "method":
                return $this->req->server["request_method"];
            case "url":
            case "origin":
                return $this->req->server["request_uri"];
            case "path":
                return isset($this->req->server["path_info"]) ? $this->req->server["path_info"] : "";
            case "query":
            case "querystring":
                return isset($this->req->server["query_string"]) ? $this->req->server["query_string"] : "";
            case "host":
            case "hostname":
                return isset($this->req->header["host"]) ? $this->req->header["host"] : "";
            case "protocol":
                return $this->req->server["server_protocol"];
            default:
                return $this->req->$name;
        }
    }
}
```