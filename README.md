# CloudWatchAutoAlarms: cree automáticamente un conjunto de alarmas de CloudWatch con etiquetado

![CloudWatchAutoAlarms Architecture Diagram](./CloudWatchAutoAlarmsArchitecture.png)

La función AWS Lambda CloudWatchAutoAlarms permite crear rápida y automáticamente un conjunto estándar de alarmas de CloudWatch para sus instancias Amazon EC2 o funciones AWS Lambda mediante etiquetas. Previene errores que pueden ocurrir al crear alarmas manualmente, reduce el tiempo necesario para implementar alarmas y reduce la falta de habilidades necesarias para crear y administrar alarmas. Puede resultar especialmente útil durante una migración grande a AWS, donde se pueden migrar muchos recursos a su cuenta de AWS a la vez.

La configuración predeterminada crea alarmas para las siguientes métricas de Amazon EC2 para instancias de Windows, Amazon Linux, Redhat, Ubuntu o SUSE EC2:
*  CPU Utilization
*  CPU Credit Balance (Para instancias de clase T)
*  Disk Space Used % (Amazon CloudWatch agent [predefined basic metric](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create-cloudwatch-agent-configuration-file-wizard.html))
*  Memory Used % (Amazon CloudWatch agent [predefined basic metric](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create-cloudwatch-agent-configuration-file-wizard.html))

La configuración predeterminada crea alarmas para la siguiente [métrica de AWS RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-metrics.html):

* CPU Utilization

Las alarmas se crean para clústeres de RDS, así como para instancias de bases de datos de RDS.

La configuración predeterminada también crea alarmas para las siguientes [métricas de AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-metrics.html#monitoring-metrics-types):

* Errors
* Throttles

Puede cambiar o agregar alarmas actualizando el diccionario **default_alarms** en [cw_auto_alarms.py](src/cw_auto_alarms.py).

Las alarmas creadas se pueden configurar para notificar un tema de Amazon SNS que especifique mediante la variable de entorno **DEFAULT_ALARM_SNS_TOPIC_ARN**. Consulte la sección **Configuración** para obtener más detalles.

Las alarmas de Amazon CloudWatch se crean cuando una instancia EC2 con la clave de etiqueta **Create_Auto_Alarms** ingresa al estado **running** y se eliminan cuando la instancia se encuentra en estado **terminated**.
Se pueden crear alarmas cuando se lanza una instancia por primera vez o después deteniéndola e iniciándola.

Las alarmas se crean y configuran en función de etiquetas EC2 que incluyen el nombre de la métrica, la comparación, el período, la estadística y el umbral.

La sintaxis del nombre de etiqueta para las métricas proporcionadas por AWS es:

AutoAlarm-\<**Namespace**>-\<**MetricName**>-\<**ComparisonOperator**>-\<**Period**>-\<**EvaluationPeriods**>-\<**Statistic**>-\<**Description**>

Donde:

* **Namespace** es el espacio de nombres de CloudWatch Alarms para la métrica. Para las métricas EC2 proporcionadas por AWS, esto es **AWS/EC2**. Para las métricas proporcionadas por el agente de CloudWatch, este es CWAgent de forma predeterminada. También puede especificar un nombre diferente como se describe en la sección **Configuración**.
* **MetricName** es el nombre de la métrica. Por ejemplo, CPUUtilization para la utilización total de CPU de EC2.
* **ComparisonOperator** es la comparación que se debe utilizar alineada con el parámetro ComparisonOperator en la acción [PutMetricData](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutMetricAlarm.html) de la API de Amazon CloudWatch.
* **Period** es el tiempo utilizado para evaluar la métrica.  Puede especificar un valor entero seguido de s para segundos, m para minutos, h para horas, d para días y w para semanas. Su período de evaluación debe respetar los límites del período de evaluación de CloudWatch.
* **EvaluationPeriods** es el número de períodos en los que se evalúa la alarma. Esta propiedad es opcional y, si se omite, el valor predeterminado es 1.
* **Statistic** es la estadística del nombre de métrica especificado, distinta del percentil.
* **Description** es la descripción de la alarma CloudWatch. Esta propiedad es opcional y, si se omite, se utiliza una descripción predeterminada.

El valor de la etiqueta se utiliza para especificar el umbral. También puede [crear alarmas para métricas personalizadas de Amazon CloudWatch](#alarming-on-custom-amazon-ec2-metrics).

Por ejemplo, una de las alarmas predeterminadas preconfiguradas que se incluyen en el diccionario **default_alarms** es **AutoAlarm-AWS/EC2-CPUUtilization-GreaterThanThreshold-5m-1-Average-Created_by_CloudWatchAutoAlarms**.
Cuando una instancia con la clave de etiqueta **Create_Auto_Alarms** ingresa al estado **running**, se creará una alarma para la métrica **CPUUtilization** CloudWatch EC2 proporcionada por AWS.
También se crearán alarmas adicionales para la instancia EC2 según la plataforma y las alarmas definidas en el diccionario de Python **default_alarms** definido en [cw_auto_alarms.py](src/cw_auto_alarms.py).

Las alarmas se pueden actualizar cambiando la clave o el valor de la etiqueta y deteniendo e iniciando la instancia.

## Requerimientos

1. Se requiere la AWS CLI para implementar la función Lambda mediante las instrucciones de implementación.
2. La AWS CLI debe configurarse con credenciales válidas para crear la pila de CloudFormation, la función lambda y los recursos relacionados. También debe tener derechos para cargar nuevos objetos en el depósito de S3 que especifique en los pasos de implementación.
3. Las instancias EC2 deben tener el agente de CloudWatch instalado y configurado con [los conjuntos de métricas predefinidas básicas, estándar o avanzadas](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create-cloudwatch-agent-configuration-file-wizard.html) para que funcionen las alarmas predeterminadas para las métricas personalizadas de CloudWatch. Se proporcionan los scripts denominados [userdata_linux_basic.sh](./userdata_linux_basic.sh), [userdata_linux_standard.sh](./userdata_linux_standard.sh), and [userdata_linux_advanced.sh](./userdata_linux_advanced.sh) para instalar y configurar el agente de CloudWatch en instancias EC2 basadas en Linux con sus respectivos conjuntos de métricas predefinidas.

## Configuración

Hay una serie de configuraciones que se pueden personalizar actualizando las variables de entorno de la función Lambda de CloudWatchAutoAlarms definidas en el template de CloudFormation denominado [CloudWatchAutoAlarms.yaml](./CloudWatchAutoAlarms.yaml).
La configuración solo afectará a las alarmas nuevas que cree, por lo que debe personalizar estos valores para cumplir con sus requisitos antes de implementar la función Lambda. La siguiente lista proporciona una descripción de la configuración junto con el nombre de la variable de entorno y el valor predeterminado:

* **ALARM_TAG**: Create_Auto_Alarms
    * La función Lambda CloudWatchAutoAlarms solo creará alarmas para instancias etiquetadas con esta etiqueta. El nombre de la etiqueta predeterminada es Create_Auto_Alarms. Si desea utilizar un nombre diferente, cambie el valor de la variable de entorno ALARM_TAG.
* **CREATE_DEFAULT_ALARMS**: true
    * Cuando es true, esto dará como resultado que se cree el conjunto de alarma predeterminado cuando la etiqueta **Create_Auto_Alarms** esté presente. Si se establece en false, las alarmas se crearán solo para las etiquetas de alarma definidas en la instancia.
* **CLOUDWATCH_NAMESPACE**: CWAgent
    * Puede cambiar el espacio de nombres donde la función Lambda debe buscar sus métricas de CloudWatch. El espacio de nombres de métricas del agente de CloudWatch predeterminado es CWAgent. Si la configuración de su agente de CloudWatch utiliza un namespace diferente, actualice esta variable de entorno.
* **CLOUDWATCH_APPEND_DIMENSIONS**: InstanceId, ImageId, InstanceType, AutoScalingGroupName
    * Puede agregar dimensiones de métricas EC2 a todas las métricas recopiladas por el agente de CloudWatch. Esta variable de entorno se alinea con su configuración de CloudWatch para [**append_dimensions**](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html#CloudWatch-Agent-Configuration-File-Metricssection).  La configuración predeterminada incluye todas las dimensiones admitidas: InstanceId, ImageId, InstanceType, AutoScalingGroupName.
* **DEFAULT_ALARM_SNS_TOPIC_ARN**:  arn:${AWS::Partition}:sns:${AWS::Region}:${AWS::AccountId}:CloudWatchAutoAlarmsSNSTopic
    * Puede definir un tema de Amazon SNS que la función Lambda especificará como destino de notificación para las alarmas creadas. Las instrucciones de implementación incluyen un topic SNS que puede implementar y utilizar con la solución. Usted proporciona el nombre de recurso de Amazon (ARN) del tema de Amazon SNS con el parámetro **AlarmNotificationARN** cuando implementa la plantilla de CloudFormation CloudWatchAutoAlarms.yaml. Si deja el valor del parámetro **AlarmNotificationARN** en blanco, esta variable de entorno no se establece y las alarmas creadas no utilizarán notificaciones. La solución también permite especificar un tema de SNS único por recurso de AWS al incluir una etiqueta con la key **`notify`** con el valor establecido en el ARN del topic SNS que debe ser el destino de las alarmas para ese recurso específico.
* **ALARM_IDENTIFIER_PREFIX**:  AutoAlarm
    * El nombre del prefijo que se agrega al principio de cada alarma de CloudWatch creada por la solución.  (por ejemplo para "AutoAlarm":  [AutoAlarm-i-00e4f327736cb077f-CPUUtilization-GreaterThanThreshold-80-5m]). Debe actualizar esta variable a través de **AlarmIdentifierPrefix** en la plantilla de CloudFormation [CloudWatchAutoAlarms.yaml](./CloudWatchAutoAlarms.yaml) para que la política de IAM se actualice a modo de alinearse con su nombre personalizado.

Puede actualizar los umbrales de las alarmas predeterminadas actualizando las siguientes variables de entorno:

   **Para alarmas de detección de anomalías**:

    ALARM_DEFAULT_ANOMALY_THRESHOLD: 2

   **Para Amazon EC2**:

    ALARM_CPU_HIGH_THRESHOLD: 75
    ALARM_CPU_CREDIT_BALANCE_LOW_THRESHOLD: 100
    ALARM_MEMORY_HIGH_THRESHOLD: 75
    ALARM_DISK_PERCENT_LOW_THRESHOLD: 20

   **Para AWS RDS**:

    ALARM_RDS_CPU_HIGH_THRESHOLD: 75

   **Para AWS Lambda**:

    ALARM_LAMBDA_ERROR_THRESHOLD: 0
    ALARM_LAMBDA_THROTTLE_THRESHOLD: 0

## Desplegar

1. Clona el repositorio github de amazon-cloudwatch-auto-alarms en tu computadora usando el siguiente comando:

       git clone https://github.com/aws-samples/amazon-cloudwatch-auto-alarms

2. Configure la AWS CLI con las credenciales para su cuenta de AWS. Este tutorial utiliza credenciales temporales proporcionadas por AWS Single Sign On mediante la opción **Línea de comando o acceso programático**. Esto establece las variables de entorno de AWS **AWS_ACCESS_KEY_ID**, **AWS_SECRET_ACCESS_KEY** y **AWS_SESSION_TOKEN** con las credenciales adecuadas para usar con la CLI de AWS.
3. Cree un topic de Amazon SNS que CloudWatchAutoAlarms utilizará para las notificaciones. Puede utilizar este template de muestra de Amazon SNS CloudFormation para crear un topic de SNS. Deje el parámetro OrganizationID en blanco; se utiliza para implementaciones de múltiples cuentas.

       aws cloudformation create-stack --stack-name amazon-cloudwatch-auto-alarms-sns-topic \
       --template-body file://CloudWatchAutoAlarms-SNS.yaml \
       --parameters ParameterKey=OrganizationID,ParameterValue="" \
       --region <enter your aws region id, e.g. "us-east-1">
4. Cree un S3 bucket que se utilizará para almacenar y acceder al paquete de implementación de la función lambda de CloudWatchAutoAlarms si no tiene uno. Puede utilizar [este template de muestra de S3 CloudFormation](./CloudWatchAutoAlarms-S3.yaml). Puede dejar el parámetro ID de AWS Organizations en blanco si esta función lambda solo se implementará en su cuenta AWS actual:

       aws cloudformation create-stack --stack-name amazon-cloudwatch-auto-alarms-s3-bucket \
        --template-body file://CloudWatchAutoAlarms-S3.yaml \
        --parameters ParameterKey=OrganizationID,ParameterValue="" \
        --region <enter your aws region id, e.g. "us-east-1">
5. Actualice las variables de entorno en el template CloudFormation de [CloudWatchAutoAlarms](./CloudWatchAutoAlarms.yaml) para configurar los ajustes predeterminados, como los umbrales de alarma.
6. Cree un archivo zip que contenga el código de función AWS Lambda de CloudWatchAutoAlarms ubicado en el directorio [src](./src). Este es el paquete de implementación que utilizará para implementar la función AWS Lambda. En una Mac, puedes usar el comando zip:

       zip -j amazon-cloudwatch-auto-alarms.zip src/*
7. Copie el archivo **amazon-cloudwatch-auto-alarms.zip** a su S3 bucket.

       aws s3 cp amazon-cloudwatch-auto-alarms.zip s3://<bucket name>

   Si creó el bucket S3 utilizando [este template de muestra de S3 CloudFormation](./CloudWatchAutoAlarms-S3.yaml) en el paso 3, puede obtener el nombre del depósito desde la consola de administración de AWS o ejecutar el siguiente comando de AWS CLI:

       aws cloudformation describe-stacks --stack-name amazon-cloudwatch-auto-alarms-s3-bucket \
       --query "Stacks[0].Outputs[?ExportName=='amazon-cloudwatch-auto-alarms-bucket-name'].OutputValue" \
       --output text \
       --region <enter your aws region id, e.g. "us-east-1">

8. Implemente la función lambda de AWS utilizando el paquete de despliegue que cargó en su bucket S3:

       aws cloudformation create-stack --stack-name amazon-cloudwatch-auto-alarms \
       --template-body file://CloudWatchAutoAlarms.yaml \
       --capabilities CAPABILITY_IAM \
       --parameters ParameterKey=S3DeploymentKey,ParameterValue=amazon-cloudwatch-auto-alarms.zip \
       ParameterKey=S3DeploymentBucket,ParameterValue=<S3 bucket with your deployment package> \
       ParameterKey=AlarmNotificationARN,ParameterValue=<SNS Topic ARN for Alarm Notifications> \
       --region <enter your aws region id, e.g. "us-east-1">

   Si no desea habilitar las notificaciones SNS, puede configurar **ParameterValue** en **""** para **AlarmNotificationARN**.

   Puede recuperar el ARN del tema SNS del paso 3 para el valor del parámetro **AlarmNotificationARN** ejecutando el siguiente comando:

       aws cloudformation describe-stacks --stack-name amazon-cloudwatch-auto-alarms-sns-topic \
       --query "Stacks[0].Outputs[?ExportName=='amazon-cloudwatch-auto-alarms-sns-topic-arn'].OutputValue" \
       --output text --region <enter your aws region id, e.g. "us-east-1">

## Activar

### Amazon EC2
Para crear el conjunto de alarmas predeterminado para una instancia de Amazon EC2 o una función AWS Lambda, simplemente necesita etiquetar la instancia EC2 o la función Lambda con la clave de etiqueta para la activación definida por la variable de entorno **ALARM_TAG**. La clave de activación de etiqueta predeterminada es **Create_Auto_Alarms**.

Para las instancias Amazon EC2, debe agregar esta etiqueta durante el lanzamiento de la instancia o puede agregar esta etiqueta en cualquier momento a una instancia y luego detenerla e iniciarla para crear el conjunto de alarmas predeterminado, así como cualquier alarma personalizada específica de la instancia.

También puede invocar manualmente la función lambda de CloudWatchAutoAlarms con la siguiente carga útil de eventos para crear/actualizar alarmas EC2 sin tener que detener e iniciar sus instancias EC2:
```json
{
  "action": "scan"
}
```
Puede hacer esto con una ejecución de prueba de la función CloudWatchAutoAlarms AWS Lambda. Abra la Consola de administración de AWS Lambda y realice una invocación de prueba desde la pestaña **Test** con la carga útil que se proporciona aquí.

El template [CloudWatchAutoAlarms.yaml](CloudWatchAutoAlarms.yaml) incluye dos reglas de eventos de CloudWatch. Se invoca la función Lambda en los estados de instancia en `running` y `terminated`. El otro invoca la función Lambda en un horario diario. El evento programado diariamente actualizará las alarmas existentes y también creará alarmas con wildcard tags.

### Amazon RDS

Para Amazon RDS, puede agregar esta etiqueta a un clúster de base de datos de RDS o a una instancia de base de datos en cualquier momento para crear el conjunto de alarmas predeterminado, así como cualquier alarma personalizada que se haya especificado como etiquetas en el clúster o instancia.

### AWS Lambda

Para AWS Lambda, puede agregar esta etiqueta a una función de AWS Lambda en cualquier momento para crear el conjunto de alarmas predeterminado, así como cualquier alarma personalizada y específica de la función.

## Soporte de notificación

Puede definir un tema de Amazon Simple Notification Service (Amazon SNS) que la función Lambda especificará como destino de notificación para las alarmas creadas. The deployment instructions include an SNS topic that you can deploy and use with the solution.  You provide the Amazon SNS Topic Amazon Resource Name (ARN) with the **AlarmNotificationARN** parameter when you deploy the CloudWatchAutoAlarms.yaml CloudFormation template.  This parameter sets the **`DEFAULT_ALARM_SNS_TOPIC_ARN`** environment variable to the ARN you specified.  If you leave the **AlarmNotificationARN** parameter value blank, then this environment variable is not set and created alarms won't use notifications.  

La solución también le permite especificar un topic SNS único por recurso de AWS configurando un tag con la key **`notify`** y el valor establecido en el ARN del tema de SNS que debe ser el destino de las alarmas para ese recurso específico. Para cualquier recurso que no tenga configurado el tag de **`notify`**, se utilizará el ARN del topic SNS predeterminado.

Puede aplicar una estrategia de etiquetado que incluya la etiqueta de **`notify`** para que grupos de recursos notifiquen sobre grupos de recursos específicos. Por ejemplo, considere una etiqueta con la clave **`Team`** y el valor **`Windows`**. Puede alinear el etiquetado de esta clave/valor específico con el tema de SNS para la compatibilidad con Windows (por ejemplo, **`notify`**: arn:aws:sns:us-east-1:123456789012:WindowsSupport)

## Cambiar el conjunto de alarma predeterminado

Puede agregar, eliminar y personalizar alarmas en el conjunto de alarmas predeterminado. Las alarmas predeterminadas se definen en el diccionario de Python **default_alarms** en [cw_auto_alarms.py](src/cw_auto_alarms.py).

Para crear una alarma, debe identificar de forma exclusiva la métrica sobre la que desea generar la alarma. Las métricas estándar de Amazon EC2 incluyen la dimensión **InstanceId** para identificar de forma única cada métrica estándar asociada con una instancia EC2. Si desea agregar una alarma basada en una métrica de instancia EC2 estándar, puede usar la sintaxis del nombre de la etiqueta: AutoAlarm-AWS/EC2-<**MetricName**>-<**ComparisonOperator**>-<**Period**>-<**EvaluationPeriods**>-<**Statistic**>-<**Description**> Esta sintaxis no incluye ningún nombre de dimensión porque la dimensión InstanceId se utiliza para métricas en el espacio de nombres AWS/EC2. Estas métricas de EC2 proporcionadas por AWS son comunes en todas las plataformas para EC2.

De manera similar, las métricas de AWS Lambda incluyen la dimensión FunctionName para identificar de forma única cada métrica estándar asociada con una función de AWS Lambda. Si desea agregar una alarma basada en una métrica estándar de AWS Lambda, puede utilizar la sintaxis del nombre de la etiqueta: AutoAlarm-AWS/Lambda-<MetricName>-<ComparisonOperator>-<Period>-<EgressionPeriods>-<Statistic>- <Descripción> Puede agregar cualquier métrica estándar de Amazon CloudWatch para Amazon EC2 o AWS Lambda al diccionario default_alarms en la clave del diccionario AWS/EC2 o AWS/Lambda utilizando esta sintaxis de etiqueta.

De manera similar, las métricas de AWS Lambda incluyen la dimensión **FunctionName** para identificar de forma única cada métrica estándar asociada con una función de AWS Lambda. Si desea agregar una alarma basada en una métrica estándar de AWS Lambda, puede utilizar la sintaxis del nombre de la etiqueta: AutoAlarm-AWS/Lambda-<**MetricName**>-<**ComparisonOperator**>-<**Period**>-<**EgressionPeriods**>-<**Statistic**>-<**Description**> Puede agregar cualquier métrica estándar de Amazon CloudWatch para Amazon EC2 o AWS Lambda al diccionario default_alarms en la clave del diccionario **AWS/EC2** o **AWS/Lambda** utilizando esta sintaxis de etiqueta.

## Soporte Wildcard para valores de dimensión en alarmas de instancia EC2

La solución le permite especificar un comodín para un valor de dimensión para crear alarmas de CloudWatch para todos los valores de dimensión. Esto es particularmente útil para crear alarmas para todas las particiones y unidades de un sistema o cuando el valor de una dimensión no se conoce o puede variar entre instancias EC2.

Por ejemplo, el agente de CloudWatch publica la métrica `disk_used_percent` para los discos conectados a una instancia EC2 de Linux. Las dimensiones de esta métrica para Amazon Linux son el `device name`, `fstype` y `path`.

La etiqueta de alarma para esta métrica está codificada en el diccionario de Python `default_alarms` en `cw_auto_alarms.py` para crear una alarma para el volumen raíz cuyas dimensiones y valores predeterminados son:

* device: nvme0n1p1
* fstype: xfs
* path: /

esto es equivalente a la siguiente etiqueta predeterminada en la solución:

```
AutoAlarm-CWAgent-disk_used_percent-device-nvme0n1p1-fstype-xfs-path-/-GreaterThanThreshold-5m-1-Average-Created_by_CloudWatchAutoAlarms
```

Si desea activar una alarma en todos los discos conectados a una instancia EC2, debe especificar el nombre del dispositivo, el tipo de sistema de archivos y los valores de dimensión de ruta para cada disco, que variarán. Cada instancia EC2 también puede tener una cantidad diferente de discos y diferentes valores de dimensión.

La solución aborda este requisito permitiéndole especificar un comodín para el valor de dimensión. Por ejemplo, la etiqueta de alarma para `disk_used_percent` para Amazon Linux especificada en el diccionario `default_alarms` cambiaría a:

```python
                {
                    'Key': alarm_separator.join(
                        [alarm_identifier, cw_namespace, 'disk_used_percent', 'device', '*', 'fstype', 'xfs', 'path',
                         '*', 'GreaterThanThreshold', default_period, default_evaluation_periods, default_statistic,
                         'Created_by_CloudWatchAutoAlarms']),
                    'Value': alarm_disk_used_percent_threshold
                },
```

Esto produce la etiqueta de alarma equivalente:

```
AutoAlarm-CWAgent-disk_used_percent-device-*-fstype-xfs-path-*-GreaterThanThreshold-5m-1-Average-Created_by_CloudWatchAutoAlarms
```

En este ejemplo, hemos especificado un comodín para las dimensiones `device` y `path`. Con este ejemplo, la solución consultará las métricas de CloudWatch y creará una alarma para cada dispositivo único y valores de dimensión de ruta para cada instancia de Amazon Linux.

Si su instancia EC2 tuviera dos discos con las siguientes dimensiones:

*Disco 1*
* device: nvme0n1p1
* fstype: xfs
* path: /

*Disco 2*
* device: nvme1n1p1
* fstype: xfs
* path: /disk2

Luego se crearían dos alarmas usando un comodín `*` para las dimensiones `device` y `path`:
* AutoAlarm-\<InstanceId>-CWAgent-disk_used_percent-device-nvme0n1p1-fstype-xfs-path-/-GreaterThanThreshold-80-5m-1p-Average-Created_by_CloudWatchAutoAlarms
* AutoAlarm-\<InstanceId>-CWAgent-disk_used_percent-device-nvme1n1p1-fstype-xfs-path-/disk2-GreaterThanThreshold-80-5m-1p-Average-Created_by_CloudWatchAutoAlarms

Para identificar los valores de dimensión, la solución consulta las métricas de CloudWatch para identificar todas las métricas que coinciden con los valores de dimensión fijos para el nombre de métrica especificado. Luego recorre en iteración las dimensiones cuyos valores se especifican como comodín para identificar los valores de dimensión específicos necesarios para la alarma.

Debido a que la solución se basa en las métricas disponibles en CloudWatch, solo funcionará después de que el agente de CloudWatch haya publicado y enviado métricas al servicio CloudWatch. Dado que la solución está diseñada para ejecutarse al iniciar la instancia, estas métricas no estarán disponibles en el primer inicio, ya que el servicio CloudWatch aún no las habrá recibido. 

Para resolver esto, debe programar la solución para que se ejecute según lo programado utilizando la carga útil `scan`:
```json 
{
"action": "scan"
}
```
Esto proporcionará tiempo suficiente para que el agente de CloudWatch publique métricas para nuevas instancias. Puede programar la frecuencia de ejecución según el período de tiempo aceptable para el cual aún no se crean alarmas basadas en comodines para nuevas instancias.

## Creación de alarmas Anomaly Detection en CloudWatch

Las alarmas de Anomaly Detection de CloudWatch se admiten mediante los operadores de comparación `LessThanLowerOrGreaterThanUpperThreshold`, `LessThanLowerThreshold` o `GreaterThanUpperThreshold`.

Cuando especifica uno de estos operadores de comparación, la solución crea una alarma de anomaly detection y utiliza el valor de la key del tag como umbral. Consulte la [documentación de CloudWatch para obtener más detalles sobre el umbral y la detección de anomalías] [documentación de CloudWatch para obtener más detalles sobre el umbral y la anomaly detection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html).

La detección de anomalías de CloudWatch utiliza modelos de aprendizaje automático basados ​​en la métrica, las dimensiones y las estadísticas elegidas. Si crea una alarma sin un modelo actual, CloudWatch Alarms crea un nuevo modelo utilizando estos parámetros de su configuración de alarma.

Para los modelos nuevos, la banda de detección de anomalías real puede tardar hasta 3 horas en aparecer en el gráfico. El nuevo modelo puede tardar hasta dos semanas en entrenarse, por lo que la banda de detección de anomalías muestra valores esperados más precisos. Consulte la documentación para obtener más detalles.

La solución incluye código comentado para crear una alarma de detección de anomalías de CloudWatch para la utilización de la CPU en el diccionario `default_alarms`:

```python
        # This is an example alarm using anomaly detection
        # {
        #     'Key': alarm_separator.join(
        #         [alarm_identifier, 'AWS/EC2', 'CPUUtilization', 'GreaterThanUpperThreshold', default_period,
        #          default_evaluation_periods, default_statistic, 'Created_by_CloudWatchAutoAlarms']),
        #     'Value': alarm_cpu_high_anomaly_detection_default_threshold
        # }
```

Puede descomentar y actualizar este código para probar la compatibilidad con la detección de anomalías. 

La solución implementa la variable de entorno `ALARM_DEFAULT_ANOMALY_THRESHOLD` como un umbral de ejemplo que puede usar para sus alarmas de detección de anomalías.

## Alarmas en métricas custom de Amazon EC2

Las métricas capturadas por el agente de Amazon CloudWatch se consideran métricas personalizadas. Estas métricas se crean en el espacio de nombres **CWAgent** de forma predeterminada. Las métricas personalizadas pueden tener cualquier cantidad de dimensiones para identificar de forma única una métrica. Además, las dimensiones de las métricas pueden recibir nombres diferentes según la plataforma subyacente de la instancia EC2.

Por ejemplo, el nombre de la métrica utilizada para medir la utilización del espacio en disco se denomina **disk_used_percent** en Linux y **LogicalDisk % Free Space** en Windows. Las dimensiones también son diferentes; en Linux también debe incluir las dimensiones **device**, **fstype** y **path** para poder identificar de forma única un disco. En Windows, debe incluir las dimensiones **objectname** e **instance**.

En consecuencia, es más difícil crear alarmas automáticamente en diferentes plataformas para métricas personalizadas de instancias EC2 de CloudWatch.

La métrica **disk_used_percent** para Linux tiene las dimensiones adicionales: **\'device', 'fstype', 'path'**. Para métricas con dimensiones personalizadas, puede incluir el nombre y el valor de la dimensión en la sintaxis de la key del tag:
AutoAlarm-\<**Namespace**>-\<**MetricName**>-\<**DimensionName-DimensionValue...**>-\<**ComparisonOperator**>-\<**Period**>-\<**EvaluationPeriods**>-\<**Statistic**>-\<**Description**>
Por ejemplo, el nombre del tag utilizado para crear una alarma para el **disk_used_percent** promedio durante un período de 5 minutos para la partición raíz en una instancia de Amazon Linux en el espacio de nombres **CWAgent** es:
**AutoAlarm-CWAgent-disk_used_percent-device-xvda1-fstype-xfs-path-/-GreaterThanThreshold-5m-1-Average-exampleDescription**
Donde la dimensión **device** tiene un valor de **xvda1**, la dimensión **fstype** tiene un valor de **xfs** y la dimensión **path** tiene un valor de **/**.

Esta sintaxis y enfoque le permiten admitir colectivamente métricas con diferentes números de dimensiones y nombres. Con esta sintaxis, puede agregar alarmas para métricas con dimensiones personalizadas a la plataforma adecuada en el diccionario **default_alarms** en [cw_auto_alarms.py](src/cw_auto_alarms.py)

También debe asegurarse de que la variable de entorno **CLOUDWATCH_APPEND_DIMENSIONS** esté configurada correctamente para garantizar que las alarmas creadas incluyan estas dimensiones. La función lambda buscará dinámicamente los valores de estas dimensiones en tiempo de ejecución.

Si el nombre de sus dimensiones utiliza el carácter separador predeterminado '-', puede actualizar la variable **alarm_separator** en [cw_auto_alarms.py](src/cw_auto_alarms.py) con un carácter separador alternativo como '~'.

## Cree una alarma específica para una instancia EC2 específica usando etiquetas

Puede crear alarmas que sean específicas de una instancia EC2 individual agregando una etiqueta a la instancia usando la sintaxis de clave de etiqueta descrita en [cambiar el conjunto de alarmas predeterminado](#changing-the-default-alarm-set). Simplemente agregue una etiqueta a la instancia al iniciarla o reinicie la instancia después de haber agregado la etiqueta. También puede actualizar los umbrales de las alarmas creadas actualizando los valores de las etiquetas, lo que hace que la alarma se actualice cuando se detiene e inicia la instancia.

Por ejemplo, para agregar una alarma para la métrica de CloudWatch **StatusCheckFailed** de Amazon EC2 para una instancia EC2 existente:
1. En la pestaña **Tags**, elija **Manage tags** y luego elija **Add tag**. Para **Key**, ingrese **AutoAlarm-AWS/EC2-StatusCheckFailed-GreaterThanThreshold-5m-1-Average-exampleDescription**. Para Valor, ingrese **1**. Elija **Save**.
2. Detenga e inicie la instancia de Amazon EC2.
3. Después de detener y reiniciar la instancia, vaya a la página **Alarmas** en la consola de CloudWatch para confirmar que se creó la alarma. Deberías encontrar una nueva alarma llamada **AutoAlarm-<instance id omitted>-StatusCheckFailed-GreaterThanThreshold-1-5m-1p-exampleDescription**.

## Crear una alarma específica para una función específica de AWS Lambda mediante etiquetas

Puede crear alarmas que sean específicas de una función individual de AWS Lambda agregando una etiqueta a la instancia utilizando la sintaxis de clave de etiqueta descrita en [cambiar el conjunto de alarmas predeterminado](#changing-the-default-alarm-set).

## Implementación en un entorno multiregión y multicuenta

Puede implementar la función lambda de CloudWatchAutoAlarms en un entorno de múltiples cuentas y múltiples regiones mediante [CloudFormation StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html).

Siga [los pasos del 1 al 7 del proceso de implementación normal](#deploy). Para los pasos 3 y 4, ingrese su ID de AWS Organizations para el parámetro **OrganizationID** en el [template de ejemplo de S3 CloudFormation](./CloudWatchAutoAlarms-S3.yaml) y el [template de ejemplo SNS CloudFormation](./CloudWatchAutoAlarms-SNS.yaml). Esto actualizará la política de recursos para permitir el acceso a todas las cuentas de su organización de AWS.

Continúe con los siguientes pasos para implementar un AWS StackSet administrado por el servicio para la función lambda CloudWatchAutoAlarms. Esto implementará la función Lambda de CloudWatchAutoAlarms en las unidades organizativas que especifique. La función lambda también se implementará automáticamente en cuentas nuevas en la organización de AWS.

1. Utilice el [template CloudWatchAutoAlarms de CloudFormation](./CloudWatchAutoAlarms.yaml) para implementar la función Lambda en varias regiones y cuentas de su organización de AWS. Este tutorial implementa un CloudFormation StackSet administrado por el servicio en la cuenta maestra de AWS Organizations. También debe especificar el ID de la cuenta donde se creó el depósito S3 de implementación para que se utilice el mismo depósito S3 en todas las implementaciones de cuentas de su organización. Utilice el siguiente comando para implementar el servicio CloudFormation StackSet administrado:

       aws cloudformation create-stack-set --stack-set-name amazon-cloudwatch-auto-alarms \
       --template-body file://CloudWatchAutoAlarms.yaml \
       --capabilities CAPABILITY_NAMED_IAM \
       --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false \
       --permission-model SERVICE_MANAGED \
       --parameters ParameterKey=S3DeploymentKey,ParameterValue=amazon-cloudwatch-auto-alarms.zip \
       ParameterKey=S3DeploymentBucket,ParameterValue=<S3 bucket with your deployment package> \
       --region <enter your aws region id, e.g. "us-east-1">

      2. Una vez creado el StackSet, puede especificar qué cuentas y regiones de AWS se debe implementar el StackSet. Para los StackSets administrados por el servicio, especifique su ID de organización de AWS o los ID de unidad organizativa de AWS para implementar la función lambda en todas las cuentas actuales y futuras dentro de ellos. Utilice el siguiente comando de AWS CLI para implementar StackSet en su organización/unidades organizativas:

       aws cloudformation create-stack-instances --stack-set-name amazon-cloudwatch-auto-alarms \
       --operation-id amazon-cloudwatch-auto-alarms-deployment-$(date | md5) \
       --deployment-targets OrganizationalUnitIds=<Enter the target OUs where the lambda function should be deployed> \
       --regions <enter the target regions where the lambda function should be deployed e.g. "us-east-1"> \
       --region <enter your aws region id, e.g. "us-east-1">

   Puede monitorear el progreso y el estado de la operación StackSet en la consola del servicio AWS CloudFormation.

   Una vez que se complete la implementación, el estado cambiará de **RUNNING** a **SUCCEEDED**.

## Seguridad

Consulte [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) para obtener más información.

## Licencia

Esta biblioteca tiene la licencia MIT-0. Para mas, ver el archivo de LICENCIA.