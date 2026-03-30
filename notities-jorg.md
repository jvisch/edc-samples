# EDC samples

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
