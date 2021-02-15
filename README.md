# Eclipse Ditto Client SDK for Golang

[![Join the chat at https://gitter.im/eclipse/ditto](https://badges.gitter.im/eclipse/ditto.svg)](https://gitter.im/eclipse/ditto?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![License](https://img.shields.io/badge/License-EPL%202.0-green.svg)](https://opensource.org/licenses/EPL-2.0)

This repository contains the Golang client SDK for [Eclipse Ditto](https://eclipse.org/ditto/).

## Creating a client & communicating with ditto

Each client instance requires configuration object, which can be created via ditto.NewConfiguration().
The

```go
config := ditto.NewConfiguration().
    WithKeepAlive(30 * time.Second). // By default keep alive is 30 seconds
    WithCredentials(&ditto.Credentials{Username: DeviceUserName, Password: DevicePassword}).
    WithBroker(MQTTBrokerLocalURI). // set the local MQTT messages broker URI
    WithConnectHandler(connectHandler). // add a handler to be notified when the connection has been successfully initiated
    WithConnectionLostHandler(connectionLostHandler) // add a handler to be notified when a connection has been lost
```

In order to create a new client you have to use ditto.NewClient(). It is advised to defer a function which will unsubscribe and disconnect.
// Create the Ditto client instance
```go
cl = ditto.NewClient(config)
defer func() {
    cl.Unsubscribe()
    cl.Disconnect()
}()
```

After creating just call Connect(), to establish a connection. This will not subscribe you however.
```go
// Connect the Ditto client instance
if err := cl.Connect(); err != nil {
    fmt.Errorf("%v", err)
    panic("cannot connect to Hub with the provided credentials") //Or some other error handling 
}
```

## Subscribing and handling messages.
Currently when you subscribe to recieve messages you will recieve ALL messages. This means that you need to check whether the message is for you or not.
In this example we use a namespaced ID formed by the namespace in which the device belongs + it's id.

``` go
func messagesHandler(requestID string, msg *protocol.Message) {
	// check that the message is for your edge thing
	if model.NewNamespacedID(msg.Topic.Namespace, msg.Topic.EntityID).String() == NamespacedID { 
       // Do something
    } else {
        fmt.Println("the message does not target our edge thing or any of its features - skipping processing")
    }
}
```

To subscribe for incoming Ditto messages, just pass the handler to the Subscribe method of the client.
```go
cl.Subscribe(messagesHandler)
```

## Worrking with features
Let us create a simple model for switching something on and off.

```go
type SwitchStatus struct {
	On bool `json:"on"`
}
```

We want our feature have a  good abstraction and be standard - so we need to define a vorto model and refer to it.
```go
const SwitchFeatureDefinitionId = "com.bosch.iot.suite.edge.services.da.items:Switch:1.0.0"
```

### Create & configure the feature instance	
```go
exampleSwitchFeature := &model.Feature{}
exampleSwitchFeature.
    WithDefinitionFrom(SwitchFeatureDefinitionId). // refer the Vorto model
    WithProperty(SwitchFeaturePropertyPathStatus, &SwitchStatus{On: false}) // add the status according to the model
```

### Create the Ditto command to modify the features of your edge gateway thing
```go
featuresModifyCommand := things.
    NewCommand(model.NewNamespacedIDFrom(thingId)). // specify which thing you will send the command to
    Twin(). // specify that the command relates to the twin Ditto channel
    Feature(featureId). // specify the ID this command relates to
    Modify(feature) // specify the payload for the modification - i.e. the feature's JSON representation
```

Wrap the Ditto command in a Ditto envelope & send it via the client.

```go
envelope := featuresModifyCommand.Message(protocol.WithResponseRequired(false))
if err := cl.Send(envelope); err != nil {
    fmt.Errorf("could not send Ditto message: %v", err)
}
```

Modify acts as a upsert - it either updates or creates features

### Removing feature property
Let asume that we have a strruct of the following type:
```go
type SwitchPlus struct {
	On bool `json:"on"`
	Extra bool `json:"extra"`
}
```

However after some time we want to remove the extre property. We can do that with the following command:
```go
propertyRemoveCommand := things.
    NewCommand(model.NewNamespacedIDFrom(thingsId)).
    Twin().
    FeatureProperty(featureId, propertyPath). // i.e: featureId: SwitchPlus, propertyPath = "status/extra"
    Delete()
```