# Service Mesh Demo

If you would like to do a Istio demo, follow the instructions below and blow some minds (if everything works). This has been tested on mac, should work on other OS too. If you are using Windows, I feel sorry for you. This demo has been tested for `istio-1.4.5` on minikube.

## Istio Setup

Instructions here are beased on official [Istio docs](https://istio.io/docs/setup/getting-started/).

1. Ensure minikube is up and running.
2. Create a folder `istio`
3. Download latest release by running: `curl -L https://istio.io/downloadIstio | sh -`
4. `cd istio-1.4.5`
5. Export bin path, run: `export PATH=$PWD/bin:$PATH`
6. Install istio: `istioctl manifest apply --set profile=demo`
7. Run the following commands to see if Istio has been installed: 

    `kubectl get svc -n istio-system`

    `kubectl get pods -n istio-system`
8. Wait for all pods to be in `Running` state.

## Deploy Sample App

Deploy the sample bookinfo app from [Istio demo](https://istio.io/docs/examples/bookinfo/). A micorservice based app that deploys 3 versions of Boofinfo Application

![Bookinfo Application](https://github.com/learnk8s/lms/blob/master/supporting_material/istio/assets/noistio.png)

Deploy the app:

1. `kubectl label namespace default istio-injection=enabled`
2. `kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml`
3. This would deploy 3 versions of reviews app v1, v2 & v3. Wait for all pods to start (can take up to 10 mins), `kubectl get pods`
4. Check all services have been deployed `kubectl get service`
5. Run the following command to see if everything has been deployed correctly:
`kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"`

6. The app is deployed correctly if the output is: `<title>Simple Bookstore App</title>`.

## Access deployed app

Website can be accessed from outside the cluster using `Istio Gateway`.

1. `kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml`
2. Confirm gateway has been created: `kubectl get gateway` 
3. Run following commands to set gatweay url

`export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')`

`export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')`

4. For minikube run:
`export INGRESS_HOST=$(minikube ip)` 

For non-minikube check docs [here](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)

5. Export Gateway URL: `export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT`
6. Check if website is running: `curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>"`
You should expect to see this: `<title>Simple Bookstore App</title>`
6. `echo ${GATEWAY_URL}`
7. Navigate to the website in borwser, use the `GATEWAY_URL`, should look something like this: `192.168.64.8:32435/productpage`

Start the demo from here, previous steps can be done before the demo.

## Demo Functionality

Once you access the productpage, you will notice that the three reviews pages have been deployed, `no stars`, `red stars` and `black stars`. You should see them in a round robin fashion when you refresh the page.

Before you can use Istio to control the Bookinfo version routing, you need to define the available versions, called subsets, in destination rules.

Run the following command:

`kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml`

Check if destination rules have been created by running: `kubectl get destinationrules`

## Apply routing rules

Navigate to the samples directory: `cd samples/bookinfo/networking/`

It is sometimes useful for people to see the file you are applying, so don't be shy and show some yaml.

### Route all traffic to v1
`kubectl apply -f virtual-service-all-v1.yaml`

Refresh the page and you will always see page with no stars

### 50% traffic to v1 & v3
`kubectl apply -f virtual-service-reviews-50-v3.yaml`

### 80% v1, 20% v2
`kubectl apply -f virtual-service-reviews-80-20.yaml`

### Route Jason to v2, else v3
`kubectl apply -f virtual-service-reviews-jason-v2-v3.yaml`

Click on login button and type `jason` in username, press login. Jason will get v2 and if you logout, it will always route to v3.

### Delay Jason by 7 sec
`kubectl apply -f virtual-service-ratings-test-delay.yaml`

Login as `jason` and you should see requests timing out to get the ratings. If you logout, you should not see any issues. Useful for testing.

### Status 500 for Jason
`kubectl apply -f virtual-service-ratings-test-abort.yaml`

Login as `jason` and you should see ratings requests return 500. 

## Prometheus

Launch the Prometheus dashboard using:

`istioctl dashboard prometheus`

Pick some params from drop down to show the results of the query.

You can run this query too: 
`istio_requests_total{destination_service="productpage.default.svc.cluster.local"}`

Details [here](https://istio.io/docs/tasks/observability/metrics/querying-metrics/).

## Grafana

You can expose Grafana dashboard using:

`kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &`

You can access the grafana ui at [http://localhost:3000/](http://localhost:3000/)

Details on setup [here](https://istio.io/docs/tasks/observability/metrics/using-istio-dashboard/). 

## Kiali

If you would like to visulaise your mesh you can use Kiali dashboard. Details [here](https://istio.io/docs/tasks/observability/kiali/).

Run the following commands:
`istioctl manifest apply --set values.kiali.enabled=true`

`istioctl dashboard kiali`

Login using `admin` `admin` ;)
