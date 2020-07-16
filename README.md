# synopsis
Build synopsis operator using Operator SDK toolkit version "v0.19.0"

## Setup
copy the blackduck, and the blackduck-connector helm charts to your folder

From the root of the folder where you have put the helm charts, 
run the following commands :

## The operator-sdk new command creates a new operator application and generates a default directory layout based on the input .

operator-sdk new synopsys-operator --api-version=blackduck.synopsys.com/v1alpha1 --kind Blackduck --type helm --helm-chart ../blackduck

## Add the blackduck-connector api using the blackduck-connector helm chart

operator-sdk add api --helm-chart ../blackduck-connector

## Generate the bundle data for the operator

operator-sdk generate bundle  -version 0.1.0




