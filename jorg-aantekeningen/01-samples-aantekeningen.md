# EDC samples

De voorbeelden zijn uitgevoerd in een WSL-omgeving met
Ubuntu 24.04.4 LTS.

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
