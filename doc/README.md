# Introduction

[Reitit](https://github.com/metosin/reitit) is a fast data-driven router for Clojure(Script).

* Simple data-driven [route syntax](./basics/route_syntax.md)
* [Route conflict resolution](./basics/route_conflicts.md)
* First-class [route data](./basics/route_data.md)
* Bi-directional routing
* [Pluggable coercion](./coercion/coercion.md) ([schema](https://github.com/plumatic/schema) & [clojure.spec](https://clojure.org/about/spec))
* Helpers for [ring](./ring/ring.html), [http](./http/interceptors.md) & [the browser](./frontend/basics.html)
* Extendable
* Modular
* [Fast](performance.md)

Modules:

* `reitit` - all bundled
* `reitit-core` - the routing core
* `reitit-ring` - a [ring router](./ring/ring.md)
* `reitit-middleware` - [common middleware](./ring/default_middleware.md) for `reitit-ring`
* `reitit-spec` [clojure.spec](https://clojure.org/about/spec) coercion
* `reitit-schema` [Schema](https://github.com/plumatic/schema) coercion
* `reitit-swagger` [Swagger2](https://swagger.io/) apidocs
* `reitit-swagger-ui` Integrated [Swagger UI](https://github.com/swagger-api/swagger-ui).
* `reitit-frontend` Tools for [frontend routing](frontend/basics.md)
* `reitit-http` http-routing with Pedestal-style Interceptors
* `reitit-interceptors` - [common interceptors](./http/default_interceptors.md) for `reitit-http`
* `reitit-sieppari` support for [Sieppari](https://github.com/metosin/sieppari) Interceptors

## Latest version

All bundled:

```clj
[metosin/reitit "0.2.3"]
```

Optionally, the parts can be required separately:

```clj
[metosin/reitit-core "0.2.3"]

;; coercion
[metosin/reitit-spec "0.2.3"]
[metosin/reitit-schema "0.2.3"]

;; ring helpers
[metosin/reitit-ring "0.2.3"]
[metosin/reitit-middleware "0.2.3"]

;; swagger-support for ring & http
[metosin/reitit-swagger "0.2.3"]
[metosin/reitit-swagger-ui "0.2.3"]

;; frontend helpers
[metosin/reitit-frontend "0.2.3"]

;; http with interceptors
[metosin/reitit-http "0.2.3"]
[metosin/reitit-interceptors "0.2.3"]
[metosin/reitit-sieppari "0.2.3"]
```

There is [#reitit](https://clojurians.slack.com/messages/reitit/) in [Clojurians Slack](http://clojurians.net/) for discussion & help.

# Examples

## Simple router

```clj
(require '[reitit.core :as r])

(def router
  (r/router
    [["/api/ping" ::ping]
     ["/api/orders/:id" ::order-by-id]]))
```

Routing:

```clj
(r/match-by-path router "/api/ipa")
; nil

(r/match-by-path router "/api/ping")
; #Match{:template "/api/ping"
;        :data {:name ::ping}
;        :result nil
;        :path-params {}
;        :path "/api/ping"}

(r/match-by-path router "/api/orders/1")
; #Match{:template "/api/orders/:id"
;        :data {:name ::order-by-id}
;        :result nil
;        :path-params {:id "1"}
;        :path "/api/orders/1"}
```

Reverse-routing:

```clj
(r/match-by-name router ::ipa)
; nil

(r/match-by-name router ::ping)
; #Match{:template "/api/ping"
;        :data {:name ::ping}
;        :result nil
;        :path-params {}
;        :path "/api/ping"}

(r/match-by-name router ::order-by-id)
; #PartialMatch{:template "/api/orders/:id"
;               :data {:name :user/order-by-id}
;               :result nil
;               :path-params nil
;               :required #{:id}}

(r/partial-match? (r/match-by-name router ::order-by-id))
; true

(r/match-by-name router ::order-by-id {:id 2})
; #Match{:template "/api/orders/:id",
;        :data {:name ::order-by-id},
;        :result nil,
;        :path-params {:id 2},
;        :path "/api/orders/2"}
```

## Ring-router

Ring-router adds support for `:handler` functions, `:middleware` and routing based on `:request-method`. It also supports pluggable parameter coercion (`clojure.spec`), data-driven middleware, route and middleware compilation, dynamic extensions and more.

```clj
(require '[reitit.ring :as ring])

(defn handler [_]
  {:status 200, :body "ok"})

(defn wrap [handler id]
  (fn [request]
    (update (handler request) :wrap (fnil conj '()) id)))

(def app
  (ring/ring-handler
    (ring/router
      ["/api" {:middleware [[wrap :api]]}
       ["/ping" {:get handler
                 :name ::ping}]
       ["/admin" {:middleware [[wrap :admin]]}
        ["/users" {:get handler
                   :post handler}]]])))
```

Routing:

```clj
(app {:request-method :get, :uri "/api/admin/users"})
; {:status 200, :body "ok", :wrap (:api :admin}

(app {:request-method :put, :uri "/api/admin/users"})
; nil
```

Reverse-routing:

```clj
(require '[reitit.core :as r])

(-> app (ring/get-router) (r/match-by-name ::ping))
; #Match{:template "/api/ping"
;        :data {:middleware [[#object[user$wrap] :api]]
;               :get {:handler #object[user$handler]}
;        :name ::ping}
;        :result #Methods{...}
;        :path-params nil
;        :path "/api/ping"}
```
