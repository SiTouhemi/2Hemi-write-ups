# 0xL4ugh web CTF



```
(ns manifesto.core

  (:require [clojure.java.io :as io]

            [clojure.core :refer [str read-string]]

            [ring.adapter.jetty :refer [run-jetty]]

            [ring.util.response :as r]

            [ring.middleware.resource :refer [wrap-resource]]

            [ring.middleware.params :refer [wrap-params]]

            [ring.middleware.session :refer [wrap-session]]

            [selmer.parser :refer [render-file]]

            [cheshire.core :as json]

            [environ.core :refer [env]]))

  

;; thread-safe stores powered by clojure atoms

(defonce server (atom nil))

(def users (atom {}))

  

;; configure selmer path

(selmer.parser/set-resource-path! (io/resource "templates"))

  

;; records

(defrecord User [username password gists])

  

;; services

(defn insert-user

  ;; clojure's multiple-arity functions are elegant and allow code reuse

  ([username password] (insert-user username password []))

  ([username password gists] (swap! users assoc username (->User username password gists))))

(defn insert-gist [username gist] (if (contains? @users username)

                                    (swap! users assoc-in [username :gists]

                                           (conj (get-in @users [username :gists]) gist)) nil))

  

;; utilities

(defn json-response [m] {:headers {"Content-Type" "application/json"}

                         :body (json/generate-string m)})

  

(:password (@users "admin"))

[(defn routes [{:keys [request-method uri session query-params form-params]}]

   (cond

     ;; index route

     (re-matches #"/" uri)

     (-> (r/response

          (render-file "index.html"

                       {:prefer (or (query-params "prefer") (session "prefer") "light")

                        :username (session "username")

                        :url uri}))

         (assoc :session (merge {"prefer" "light"} session query-params)))

  

     ;; display user gists, protected for now

     (re-matches #"/gists" uri)

     (cond (not= (session "username") "admin")

           (json-response {:error "You do not have enough privileges"})

  

           (= request-method :get)

           (r/response

            (render-file "gists.html"

                         {:prefer (session "prefer")

                          :username (session "username")

                          :gists (get-in @users [(session "username") :gists])

                          :url uri}))

  

           (= request-method :post)

           (let [{:strs [gist]} form-params]

             ;; clojure has excellent error handling capabilities

             (try

               (insert-gist (session "username") (read-string gist))

               (r/redirect "/gists")

               (catch Exception _ (json-response {:error "Something went wrong..."}))))

  

           :else

           (json-response {:error "Something went wrong..."}))

  

     ;; login route

     (re-matches #"/login" uri)

     (cond

       (session "username")

       (r/redirect "/")

  

       (= request-method :get)

       (r/response

        (render-file "login.html"

                     {:prefer (session "prefer")

                      :user (@users (session "username"))

                      :url uri}))

       (= request-method :post)

       (let [{:strs [username password]} form-params]

         (cond

           (empty? (remove empty? [username password]))

           (json-response

            {:error "Missing fields"

             :fields (filter #(empty? (form-params %)) ["username" "password"])})

           :else

           ;; get user by username

           (let [user (@users username)]

             ;; check password

             (if (and user (= password (:password user)))

               ;; login

               (-> (r/redirect "/gists")

                   (assoc :session

                          (merge session {"username" username})))

               ;; invalid username or password

               (json-response {:error "Invalid username or password"})))))

       :else (json-response {:error "Unknown method"}))

  

     ;; logout route

     (re-matches #"/logout" uri)

     (-> (r/redirect "/") (assoc :session {}))

  

     ;; detect trailing slash java interop go brr

     (.endsWith uri "/")

     ;; remove trailing slash thread-last macro go brr

     (r/redirect (->> uri reverse rest reverse (apply str)))

  

     ;; catch all

     :else

     (-> (r/response "404 Not Found")

         (r/status 404))))

  

 ;; define app and apply middleware

 (def app (-> routes

              (wrap-resource "public")

              (wrap-params)

              (wrap-session {:cookie-name "session" :same-site :strict})))]

  

;; server utilities

(defn start-server []

  (reset! server (run-jetty (fn [req] (app req))

                            {:host (or (env :clojure-host) "0.0.0.0")

                             :port (Integer/parseInt (or (env :clojure-port) "8080"))

                             :join? false})))

  

(defn stop-server []

  (when-some [s @server]

    (.stop s)

    (reset! server nil)))

  

;; convenience repl shortcuts

(comment

  (start-server)

  (stop-server))

  

;; initialize

  

(defn -main []

  ((do (insert-user "admin" (str (random-uuid)))

       (insert-gist "admin" "self-reminder #1: with clojure, you get to closure")

       (insert-gist "admin" "self-reminder #2: clojure gives me composure")

       (insert-gist "admin" "self-reminder #3: i 💖 clojure")

       start-server)))
```

First, change the theme, intercept the request, and modify it to `?username=admin`.

Next, access the gists as an admin. The source code doesn’t sanitize input in the Clojure code, so to read the flag, simply inject the following command:

```
#=(slurp "/proc/self/environ")
```

note: The flag is and environment varianble
