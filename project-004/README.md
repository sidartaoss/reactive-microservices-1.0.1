## Message-Based Microservices

### Resilience

To illustrate this, let's modify the Hello microservice to inject failures and misbehaviors.

This new start method randomly selects one of the three strategies: (1) reply with an explicit failure, (2) forget to reply (leading to a timeout on the consumer side), or (3) send the correct result.


      @Override
      public void start() {
            vertx.eventBus().<String>consumer("hello", message -> {
                double chaos = Math.random();
                JsonObject json = new JsonObject()
                    .put("served-by", this.toString());

                if (chaos < 0.6) {
                    // Normal behavior
                    if (message.body().isEmpty()) {
                        message.reply(json.put("message", "hello"));
                    } else {
                      message.reply(json.put("message", "hello " + message.body()));
                    }
                } else if (chaos < 0.9) {
                    System.out.println("Returning a failure");
                    // Reply with a failure
                    message.fail(500, "message processing failure");
                } else {
                    System.out.println("Not replying");
                    // Just do not reply, leading to a timeout on the consumer side.
                }
            });
      }


Firstly, we should run the following script file:

      
      ./cleanup.sh


### Cleanup Script file


    #!/usr/bin/env bash
    echo "Cleaning up fat JARs"

    echo ""

    echo "Cleaning up fat JAR from project-004"
    cd ../project-004
    rm -rf project-004-1.0-SNAPSHOT.jar
    echo "Ok"

    echo ""

    echo "Cleaning up fat JAR from project-005"
    cd ../project-005
    rm -rf project-005-1.0-SNAPSHOT.jar
    echo "Ok"

    echo ""

    echo "Cleaning up current Kubernetes resources"

    echo ""

    echo "Cleaning up current Pods and Deployments"
    kubectl delete deployment --all

    echo ""

    echo "Cleaning up current Services"
    kubectl delete svc project004 project005

    echo ""

    echo "Well done!"


### Repackaging

Then, let's repackage to Kubernetes and restart the two instances of the Hello microservice.


      kubectl apply -f deployment.yaml


### Deployment Yaml file

      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: project004
      spec:
        replicas: 2
        selector:
          matchLabels:
            app: project004
        template:
          metadata:
            labels:
              app: project004
          spec:
            containers:
              - name: project004
                image: sidartasilva/project004
                imagePullPolicy: Always
                ports:
                - containerPort: 5701
                - containerPort: 8080

      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: project004
      spec:
        type: LoadBalancer
        selector:
          app: project004
        ports:
        - name: hazelcast
          port: 5701
        - name: app
          port: 8080

With this fault injection in place, we need to improve the fault-tolerance of our consumer. Indeed, the consumer may get a timeout or receive or receive an explicit failure.  

So, in the hello consumer microservice, we must change how we invoke the hello service.


