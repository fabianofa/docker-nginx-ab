
# docker-nginx-ab
Nginx docker image configured to run as A/B Tester between two other servers.

# Requirements

* Docker (https://docs.docker.com/install/linux/docker-ce/ubuntu/)
* Docker-compose https://docs.docker.com/compose/install/

# Instructions

This image was made to using with: https://github.com/fabianofa/docker-php-devenv, thus the same bridge network must be created. You may change the name and IP range. For safety reasons and to prevent users from selecting a version, it is recommended to keep only nginx running at a public port.

```
$ docker network create --subnet=10.253.254.0/24 php-apache
```

# Configuration file explained

```
nginx.conf

    ...

    # Address for A version (PHP 7.1)
    # This may be replaced by docker gateway IP if Apache server is running on host.
    # Otherwise, if it is another docker container, use its IP and port (if in the same bridge network) or host port.
    upstream version_a {
        server 10.253.254.71:80;
    }

    # Address for A version (PHP 7.3)
    upstream version_b {
        server 10.253.254.73:80;
    }

    # Effectively splitting clients between versions.
    # In this case, 20% of the traffic will go to version_a while 80% will go to version_b
    split_clients "app${remote_addr}${http_user_agent}${date_gmt}" $appversion {
        20% version_a;
        *   version_b;
    }

    # A cookie must be stored to make sure the user keeps running the same version
    # as long as he/she interact with the website, otherwise users may have different
    # behaviour for each request if A and B are considerably different.
    map $cookie_split_test_version $upstream_group {
        default $appversion;
        "version_a" "version_a";
        "version_b" "version_b";

    }

    # Setting the cookie with Max-Age will ensure that after a time, user may use the other
    # version again. But if it is sent every request, the user will never reach Max-Age.
    # This map instruction is used to make sure that, if the user is new or the cookie
    # has expired, a new cookie is added, otherwhise, nothing is send at the Response header.
    map $cookie_split_test_version $ck_content {
       default "split_test_version=$upstream_group;Path=/;Max-Age=518400;";
       "version_a" "";
       "version_b" "";
    }

        server {
        listen 80;

        client_max_body_size 64M;

        # You may add specific rules and folders that have to use the version segmentation.
        # You can also uso just / to apply to all directories.
        
        #location /example.com/ {

            # Sets Cookie header with the content that might be empty or cookie code,
            # version and Max-Age

            #add_header Set-Cookie $ck_content;
            #proxy_set_header Host $host;


            # If you use more than just $cookie_split_test_version to decide which version
            # the user should see, you may add them here. However keep in mind this article: If Is Evil
            # https://www.nginx.com/resources/wiki/start/topics/depth/ifisevil/
            # if ($upstream_group = "version_a") {
            #     proxy_pass http://10.253.254.71:80;
            #     break;
            # }
            # if ($upstream_group = "version_b") {
            #     proxy_pass http://10.253.254.73:80;
            #     break;
            # }

            #proxy_pass http://$appversion;
        #}
        
        location / {
            # Sets Cookie header with the content that might be empty or cookie code,
            # version and Max-Age
            add_header Set-Cookie $ck_content;
            proxy_set_header Host $host;

            # $upstream_group is defined by the existance of its own value in cookie or
            # by the probabilistic definition of $appversion.
            proxy_pass http://$upstream_group;
            break;
        }
    }
```
If you use SSL (and you should), make sure you make changes in `.withssl` file. Besides the extra lines defining cert files and redirects when not `https` all the splitting and cookie decision remains the same.


O que define a alternância entre os acessos está definido na linha: `"app${remote_addr}${http_user_agent}${date_gmt}"`. Todos
são variáveis do nginx, mas respectivamente se utiliza a string app, com o IP do usuário, o user-agent do usuário e a data corrente
da solicitação.

---

# References:

https://www.freecodecamp.org/news/a-b-testing-with-nginx-in-40-lines-of-code-d4f94397130a/
https://www.nginx.com/resources/wiki/start/topics/depth/ifisevil/
