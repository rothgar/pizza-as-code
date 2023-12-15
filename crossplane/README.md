# Ordering pizza with crossplane

You will need to use the crossplane [provider-pizza](https://github.com/grantgumina/provider-pizza) with a Kubernetes cluster version 1.21 or earlier. The provider-pizza is not compatible with Kubernetes 1.22 or later.The CRDs have not been updated for the `apiextensions.k8s.io/v1` spec. The provider-pizza is a proof of concept and is not intended for production use.

Create a local kind cluster and install crossplane:

```sh
echo '---
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.21.14
  - role: worker
    image: kindest/node:v1.21.14'
> kind.yaml

kind create cluster --config kind.yaml

helm repo add crossplane-stable https://charts.crossplane.io/crossplane-stable
helm repo update
helm install crossplane \
--namespace crossplane-system \
--create-namespace crossplane-stable/crossplane
```

Clone the repository and install the provider-pizza:

```sh
git clone git@github.com:grantgumina/provider-pizza.git
cd provider-pizza
make run
```

This will install the CRDs into the cluster and run the controller locally.
You should see log output in your console.

Create a provider config and order:

```sh
# This is the sample config but doesn't matter
# because you can pay with cash instead of a credit card
kubectl apply -f https://raw.githubusercontent.com/grantgumina/provider-pizza/master/examples/provider/pizza-config.yaml

echo 'apiVersion: order.provider-pizza.crossplane.io/v1alpha1
kind: Order
metadata:
name: my-order
spec:
forProvider:
placeOrder: true
address:
street: "$LOCAL_DOMINOS_ADDRESS"
      city: "$LOCAL_DOMINOS_CITY"
region: "$LOCAL_DOMINOS_STATE"
      postalCode: "$LOCAL_DOMINOS_ZIP"
phone: "$LOCAL_DOMINOS_PHONE"
    customer:
      firstName: "$YOUR_FIRST_NAME"
lastName: "$YOUR_LAST_NAME"
      email: "$YOUR_EMAIL"
serviceMethod: "Carryout"
paymentMethod: "Cash"
products: # Provider only supports a large cheese pizza or coke # https://github.com/grantgumina/provider-pizza/blob/76f58c86e8ef4380b6152393113483db1610eaf6/pkg/controller/order/order.go#L211-L221 - name: "Cheese Pizza"
providerConfigRef:
name: example' > order.yaml

kubectl apply -f order.yaml
```

Now you should be able to see an order in Kubernetes

```sh
kubectl get orders
NAME ORDER STATUS PLACED PRICE
my-order Makeline true 19.38
```

The order status will update as the order is updated from the dominos API.

```sh
NAME ORDER STATUS PLACED PRICE
my-order Oven true 19.38
```
