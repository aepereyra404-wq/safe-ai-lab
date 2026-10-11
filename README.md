openshell provider profile import -f ollama-provider.yaml \
openshell provider create --name pi-ollama --type pi-ollama --from-existing
openshell sandbox create --name pi-container --from ghcr.io/nvidia/openshell-community/sandboxes/pi:latest --policy ./ollama-policy.yaml --provider pi-ollama  -- pi


openshell sandbox upload pi-container ./models.json /sandbox/models.json
openshell sandbox upload pi-container ./settings.json /sandbox/settings.json

Then connect to the sandbox:
bash

openshell sandbox connect pi-container

The files are initially uploaded to /sandbox, so copy them into Pi's configuration directory from inside the sandbox:
bash

mkdir -p ~/.pi/agent
cp /sandbox/models.json ~/.pi/agent/models.json
cp /sandbox/settings.json ~/.pi/agent/settings.json
