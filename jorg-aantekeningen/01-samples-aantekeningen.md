# EDC samples

De voorbeelden zijn uitgevoerd in een WSL-omgeving met
Ubuntu 24.04.4 LTS.

Om de api te testen gebruik Bruno <https://www.usebruno.com/>

## Basic 1, 2 en 3

Debug run (`--debug-jvm`)
`./gradlew :basic:basic-02-health-endpoint:run --debug-jvm`

Configuratie meegeven, via `-D` (system flags voor java). Maar dan moet
je ook de gradle build file aanpassen met onderstaande. Zie
[`./build.gradle.kts`](./build.gradle.kts)

    tasks.withType<JavaExec>().configureEach { 
        systemProperties(gradle.startParameter.systemPropertiesArgs) 
    }

`./gradlew -Dedc.fs.config=/etc/eclipse/dataspaceconnector/config.properties basic:basic-03-configuration:run`

## Transfer

## Transfer 00 basis

De configuratie meegeven, let op dat de working directory
`./transfer/transfer-00-prerequisites/connector` is en de resources een
niveau lager, dus meegeven
`../resources/configuration/provider-configuration.properties`

Provider:

-   `./gradlew transfer:transfer-00-prerequisites:connector:run -Dedc.fs.config=../resources/configuration/provider-configuration.properties`

Consumer:

-   `./gradlew transfer:transfer-00-prerequisites:connector:run -Dedc.fs.config=../resources/configuration/consumer-configuration.properties`

## Transfer 01 Negotiation

1.  Maak een "asset" (bestand/waarde/iets) aan op de provider

        curl -d @transfer/transfer-01-negotiation/resources/create-asset.json \
        -H 'content-type: application/json' http://localhost:19193/management/v3/assets \
        -s | jq

    het bestand `create-asset.json` wordt met een `POST` opgestuurd. De
    response is een json-string, met `jq` wordt dat wat netter gemaakt

2.  Maak een "policy" aan op de provider, deze is zo gemaakt dat alles
    op te halen is.

        curl -d @transfer/transfer-01-negotiation/resources/create-policy.json \
        -H 'content-type: application/json' http://localhost:19193/management/v3/policydefinitions \
        -s | jq

    De policy staat in het bestand `create-policy.json` (zie
    <https://www.w3.org/TR/odrl-model/#policy-set>)

3.  Maak een "contract definition" aan op de provider.

        curl -d @transfer/transfer-01-negotiation/resources/create-contract-definition.json \
        -H 'content-type: application/json' http://localhost:19193/management/v3/contractdefinitions \
        -s | jq

    Het contract staat in `create-contract-definition.json`.

4.  Ophalen van een "catalog" door een consumer op de provider.

        curl -X POST "http://localhost:29193/management/v3/catalog/request" \
        -H 'Content-Type: application/json' \
        -d @transfer/transfer-01-negotiation/resources/fetch-catalog.json -s | jq

    Hier wordt de consumer aangeroepen, die moet dan weer een catalog op
    halen van de provider. Dit ging een eerste keer mis, na herhalen van
    de eerdere stappen ging het wel goed. Wel de provider en consumer
    opnieuw opstarten om tijdelijk geheugen te wissen.

5.  De consumer een "negotiate a contract" laten doen.

        curl -d @transfer/transfer-01-negotiation/resources/negotiate-contract.json \
        -X POST -H 'content-type: application/json' http://localhost:29193/management/v3/contractnegotiations \
        -s | jq

    In het bestand `negotiate-contract.json` moet de
    `{{contract-offer-id}}` vervangen worden met de waarde die bij het
    opvragen van de catalog (stap 4) staat onder node `odrl:hasPolicy`
    bij `@id`. Voorbeeld (deel van response bij stap 4)

    ``` json
    {
        "@id": "12092b02-e262-4b55-aa58-dcc79092a053",
        "@type": "dcat:Catalog",
        "dcat:dataset": {
        "@id": "assetId",
        "@type": "dcat:Dataset",
        "odrl:hasPolicy": {
            "@id": "MQ==:YXNzZXRJZA==:NmY1ODY3OGItMzc4Yy00MGVjLTgyZjItMjdkNTBkNGY0NGVm",     <--------
            "@type": "odrl:Offer",
            "odrl:permission": [],
            "odrl:prohibition": [],
            "odrl:obligation": []
        },
        "dcat:distribution": [
            {
    ```

    In de response staat het "contract agreement id", voorbeeldresponse:

    ``` json
    {
        "@type": "IdResponse",
        "@id": "3166d8fb-689e-471d-85ae-067ab2215c57",     <--------
        "createdAt": 1774938199984,
        "@context": {
            "@vocab": "https://w3id.org/edc/v0.0.1/ns/",
            "edc": "https://w3id.org/edc/v0.0.1/ns/",
            "odrl": "http://www.w3.org/ns/odrl/2/"
        }
    }
    ```

6.  Het "contract agreement id" ophalen

        curl -X GET "http://localhost:29193/management/v3/contractnegotiations/{{contract-negotiation-id}}" \
        --header 'Content-Type: application/json' \
        -s | jq

    met ingevuld `contract-negotiation-id`:

        curl -X GET "http://localhost:29193/management/v3/contractnegotiations/3166d8fb-689e-471d-85ae-067ab2215c57" \
        --header 'Content-Type: application/json' \
        -s | jq

    De response geeft aan dat er een succesvolle *onderhandeling* plaats
    heeft gevonden en dat de transfer plaats kan vinden. De connectors
    staan nu in de state 'transfer'. Voorbeeldresponse hieronder, met
    het `contractAgreementId` kan de transfer verder uitgevoerd worden.

    ``` json
    {
    "@type": "ContractNegotiation",
    "@id": "3166d8fb-689e-471d-85ae-067ab2215c57",
    "type": "CONSUMER",
    "protocol": "dataspace-protocol-http",
    "state": "FINALIZED",
    "counterPartyId": "provider",
    "counterPartyAddress": "http://localhost:19194/protocol",
    "callbackAddresses": [],
    "createdAt": 1774938199984,
    "assetId": "assetId",
    "contractAgreementId": "2c71a434-d3f4-4b1e-842e-dae7aa28f521",   <---------
    "correlationId": "97792ff8-c7e9-45f7-b521-03f882ab0c88",
    "@context": {
        "@vocab": "https://w3id.org/edc/v0.0.1/ns/",
        "edc": "https://w3id.org/edc/v0.0.1/ns/",
        "odrl": "http://www.w3.org/ns/odrl/2/"
    }
    }
    ```

## Transfer 02 Provider push (http transfer flow)

1.  start een httpserver (een http request logger)

        docker build -t http-request-logger util/http-request-logger
        docker run -p 4000:4000 http-request-logger

    De Dockerfile staat in de map 'samples/util/http-request-logger',
    kan je vanaf de commmandline (workdir ./samles/ aanroepen). Als je
    in WSL2 werkt, moet je in docker (de applicatie) aangeven dat er een
    integratie is met docker de wsl-image waarbinnen je werkt. En op
    linux werkt docker onder `sudo`, dus dat moet je er wel voor typen.

    Dit print http request (op poort 4000) naar de terminal.

2.  start de transfer (vanuit de consumer)

        curl -X POST "http://localhost:29193/management/v3/transferprocesses" \
        -H "Content-Type: application/json" \
        -d @transfer/transfer-02-provider-push/resources/start-transfer.json \
        -s | jq

    In de body van de json heb je het {{contract-agreement-id}} van
    stap 6 van vorige deel (transfor 01). In de Bruno omgeving (de api
    testomgeving) wordt dat automatisch gedaan met uitlezen van de
    variabelen uit de response.

    Deze transfer zorgt ervoor dat de data uit de *asset* (stap 01.1)
    naar de opgeven url gaat. In het json-bericht van de call hierboven
    staat dat de `dataDestination` is
    `http://localhost:4000/api/consumer/store`.

3.  check transfer status

        curl http://localhost:29193/management/v3/transferprocesses/<transfer process id>

    Er is een statemachine die de transfer beschrijft zie
    <https://eclipse-edc.github.io/documentation/for-adopters/control-plane/#transfer-process-states>

    De request (en push) wordt heel snel afgehandeld, dus waarschijnlijk
    krijg je alleen de status `COMPLETED` te zien.

    In de terminal waar de docker in is gestart, zal de data nu
    afgedrukt worden. In de asset uit stap 01.1 staat dat de baseUrl
    gelijk is aan `https://jsonplaceholder.typicode.com/users` en de
    data uit die url wordt naar de consumer gepusht. Het proces is heel
    snel, daarom zul je bijna alleen maar COMPLETED zien. Zet de
    http-logger uit, je zult dan zien dat na de transfer aanvraag de
    status op STARTED staat. Na enige tijd gaat 'ie naar TERMINATED,
    tenzij je wat eerder de logger weer start, dan zie je de data wat
    later binnenkomen.

## Transfer 03 Implement a simple "Consumer Pull" Http transfer flow

1.  Er een simple consumer gemaakt met een get interface

2.  het voorbeeld runnen

    Let op, eerst de eerder opgstarte consumer en provider stoppen.

    -   eerst bouwen
        `./gradlew transfer:transfer-03-consumer-pull:provider-proxy-data-plane:build`

    -   dan runnen:

        consumer (gelijk aan stap 00)

            ./gradlew transfer:transfer-00-prerequisites:connector:run \
                      -Dedc.fs.config=../resources/configuration/consumer-configuration.properties

        provider (met config uit transfer 03)

            ./gradlew transfer:transfer-03-consumer-pull:provider-proxy-data-plane:run \
                      -Dedc.fs.config=../resources/configuration/provider.properties
