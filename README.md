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

You can apply a tagging strategy that includes the **`notify`** tag for groups of resources to notify on specific groups of resources.  For example, consider a tag with key **`Team`** and value **`Windows`**.  You could align tagging of this specific key / value with the SNS topic for Windows support (e.g. **`notify`**: arn:aws:sns:us-east-1:123456789012:WindowsSupport)

## Changing the default alarm set

You can add, remove, and customize alarms in the default alarm set.  The default alarms are defined in the **default_alarms** python dictionary in [cw_auto_alarms.py](src/cw_auto_alarms.py).

In order to create an alarm, you must uniquely identify the metric that you want to alarm on.  Standard Amazon EC2 metrics include the **InstanceId** dimension to uniquely identify each standard metric associated with an EC2 instance.  If you want to add an alarm based upon a standard EC2 instance metric, then you can use the tag name syntax:
AutoAlarm-AWS/EC2-\<**MetricName**>-\<**ComparisonOperator**>-\<**Period**>-\<**EvaluationPeriods**>-\<**Statistic**>-\<**Description**>
This syntax doesn't include any dimension names because the InstanceId dimension is used for metrics in the **AWS/EC2** namespace.  These AWS provided EC2 metrics are common across all platforms for EC2.

Similarly, AWS Lambda metrics include the **FunctionName** dimension to uniquely identify each standard metric associated with an AWS Lambda function.  If you want to add an alarm based upon a standard AWS Lambda metric, then you can use the tag name syntax:
AutoAlarm-AWS/Lambda-\<**MetricName**>-\<**ComparisonOperator**>-\<**Period**>-\<**EvaluationPeriods**>-\<**Statistic**>-\<**Description**>
You can add any standard Amazon CloudWatch metric for Amazon EC2 or AWS Lambda into the **default_alarms** dictionary under the **AWS/EC2** or **AWS/Lambda** dictionary key using this tag syntax.

## Wildcard support for dimension values on EC2 instance alarms

La solución le permite especificar un comodín para un valor de dimensión para crear alarmas de CloudWatch para todos los valores de dimensión. Esto es particularmente útil para crear alarmas para todas las particiones y unidades de un sistema o cuando el valor de una dimensión no se conoce o puede variar entre instancias EC2.

For example, the CloudWatch agent publishes the `disk_used_percent` metric for disks attached to a Linux EC2 instance.  The dimensions for this metric for Amazon Linux are `device name`, `fstype`, and `path`.

The alarm tag for this metric is hardcoded in the `default_alarms` python dictionary in `cw_auto_alarms.py` to create an alarm for the root volume whose default dimensions and values are: 

* device: nvme0n1p1
* fstype: xfs
* path: /

esto es equivalente a la siguiente etiqueta predeterminada en la solución:

```
AutoAlarm-CWAgent-disk_used_percent-device-nvme0n1p1-fstype-xfs-path-/-GreaterThanThreshold-5m-1-Average-Created_by_CloudWatchAutoAlarms
```

Si desea activar una alarma en todos los discos conectados a una instancia EC2, debe especificar el nombre del dispositivo, el tipo de sistema de archivos y los valores de dimensión de ruta para cada disco, que variarán. Cada instancia EC2 también puede tener una cantidad diferente de discos y diferentes valores de dimensión.

The solution addresses this requirement by allowing you to specify a wildcard for the dimension value.  For example, the Alarm tag for `disk_used_percent` For Amazon Linux specified in the `default_alarms` dictionary would change to:

```python
                {
                    'Key': alarm_separator.join(
                        [alarm_identifier, cw_namespace, 'disk_used_percent', 'device', '*', 'fstype', 'xfs', 'path',
                         '*', 'GreaterThanThreshold', default_period, default_evaluation_periods, default_statistic,
                         'Created_by_CloudWatchAutoAlarms']),
                    'Value': alarm_disk_used_percent_threshold
                },
```

This yields the equivalent alarm tag:

```
AutoAlarm-CWAgent-disk_used_percent-device-*-fstype-xfs-path-*-GreaterThanThreshold-5m-1-Average-Created_by_CloudWatchAutoAlarms
```

In this example, we have specified a wildcard for the `device` and `path` dimensions.  Using this example, the solution will query CloudWatch metrics and create an alarm for each unique device and path dimension values for each Amazon Linux instance.  

Si su instancia EC2 tuviera dos discos con las siguientes dimensiones:

*Disco 1*
* device: nvme0n1p1
* fstype: xfs
* path: /

*Disco 2*
* device: nvme1n1p1
* fstype: xfs
* path: /disk2

Then two alarms would be created using a `*` wildcard for the `device` and `path` dimensions:
* AutoAlarm-\<InstanceId>-CWAgent-disk_used_percent-device-nvme0n1p1-fstype-xfs-path-/-GreaterThanThreshold-80-5m-1p-Average-Created_by_CloudWatchAutoAlarms
* AutoAlarm-\<InstanceId>-CWAgent-disk_used_percent-device-nvme1n1p1-fstype-xfs-path-/disk2-GreaterThanThreshold-80-5m-1p-Average-Created_by_CloudWatchAutoAlarms


Para identificar los valores de dimensión, la solución consulta las métricas de CloudWatch para identificar todas las métricas que coinciden con los valores de dimensión fijos para el nombre de métrica especificado. Luego recorre en iteración las dimensiones cuyos valores se especifican como comodín para identificar los valores de dimensión específicos necesarios para la alarma.

Debido a que la solución se basa en las métricas disponibles en CloudWatch, solo funcionará después de que el agente de CloudWatch haya publicado y enviado métricas al servicio CloudWatch. Dado que la solución está diseñada para ejecutarse al iniciar la instancia, estas métricas no estarán disponibles en el primer inicio, ya que el servicio CloudWatch aún no las habrá recibido. 

In order to resolve this, you should schedule the solution to run on schedule using the `scan` payload:
```json 
{
"action": "scan"
}
```

Esto proporcionará tiempo suficiente para que el agente de CloudWatch publique métricas para nuevas instancias. Puede programar la frecuencia de ejecución según el período de tiempo aceptable para el cual aún no se crean alarmas basadas en comodines para nuevas instancias.


## Creación de alarmas Anomaly Detection en CloudWatch

CloudWatch Anomaly Detection Alarms are supported using the comparison operators `LessThanLowerOrGreaterThanUpperThreshold`, `LessThanLowerThreshold`, or `GreaterThanUpperThreshold`.

When you specify one of these comparison operators, the solution creates an anomaly detection alarm and uses the value for the tag key as the threshold.  Refer to the [CloudWatch documentation for more details on the threshold and anomaly detection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html).

La detección de anomalías de CloudWatch utiliza modelos de aprendizaje automático basados ​​en la métrica, las dimensiones y las estadísticas elegidas. Si crea una alarma sin un modelo actual, CloudWatch Alarms crea un nuevo modelo utilizando estos parámetros de su configuración de alarma.
Para los modelos nuevos, la banda de detección de anomalías real puede tardar hasta 3 horas en aparecer en el gráfico. El nuevo modelo puede tardar hasta dos semanas en entrenarse, por lo que la banda de detección de anomalías muestra valores esperados más precisos. Consulte la documentación para obtener más detalles.

The solution includes commented out code for creating a CloudWatch Anomaly Detection Alarm for CPU Utilization in the `default_alarms` dictionary: 

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

The solution implements the environment variable `ALARM_DEFAULT_ANOMALY_THRESHOLD` as an example threshold you can use for your anomaly detection alarms.

## Alarming on custom Amazon EC2 metrics

Metrics captured by the Amazon CloudWatch agent are considered custom metrics.  These metrics are created in the **CWAgent** namespace by default.  Custom metrics may have any number of dimensions in order to uniquely identify a metric.  Additionally, the metric dimensions may be named differently based upon the underlying platform for the EC2 instance.

For example, the metric name used to measure the disk space utilization is named **disk_used_percent** in Linux and **LogicalDisk % Free Space** in Windows.  The dimensions are also different, in Linux you must also include the **device**, **fstype**, and **path** dimensions in order to uniquely identify a disk.  In Windows, you must include the **objectname** and **instance** dimensions.

Consequently, it is more difficult to automatically create alarms across different platforms for custom CloudWatch EC2 instance metrics.

The **disk_used_percent** metric for Linux has the additional dimensions:  **\'device', 'fstype', 'path'**.  For metrics with custom dimensions, you can include the dimension name and value in the tag key syntax:
AutoAlarm-\<**Namespace**>-\<**MetricName**>-\<**DimensionName-DimensionValue...**>-\<**ComparisonOperator**>-\<**Period**>-\<**EvaluationPeriods**>-\<**Statistic**>-\<**Description**>
For example, the tag name used to create an alarm for the average **disk_used_percent** over a 5 minute period for the root partition on an Amazon Linux instance in the **CWAgent** namespace is:
**AutoAlarm-CWAgent-disk_used_percent-device-xvda1-fstype-xfs-path-/-GreaterThanThreshold-5m-1-Average-exampleDescription**
Where the **device** dimension has a value of **xvda1**, the **fstype** dimension has a value of **xfs**, and the **path** dimension has a value of **/**.

This syntax and approach allows you to collectively support metrics with different numbers of dimensions and names.  Using this syntax, you can add alarms for metrics with custom dimensions to the appropriate platform in the **default_alarms** dictionary in [cw_auto_alarms.py](src/cw_auto_alarms.py)

You should also make sure that the **CLOUDWATCH_APPEND_DIMENSIONS** environment variable is set correctly in order to ensure that created alarms include these dimensions.  The lambda function will dynamically lookup the values for these dimensions at runtime.

If your dimensions name uses the default separator character '-', then you can update the **alarm_separator** variable in [cw_auto_alarms.py](src/cw_auto_alarms.py) with an alternative seperator character such as '~'.

## Create a specific alarm for a specific EC2 instance using tags

You can create alarms that are specific to an individual EC2 instance by adding a tag to the instance using the tag key syntax described in [changing the default alarm set](#changing-the-default-alarm-set).  Simply add a tag to the instance on launch or restart the instance after you have added the tag.  You can also update the thresholds for created alarms by updating the tag values, causing the alarm to be updated when the instance is stopped and started.

For example, to add an alarm for the Amazon EC2 **StatusCheckFailed** CloudWatch metric for an existing EC2 instance:
1. On the **Tags** tab, choose **Manage tags**, and then choose **Add tag**. For **Key**, enter **AutoAlarm-AWS/EC2-StatusCheckFailed-GreaterThanThreshold-5m-1-Average-exampleDescription**. For Value, enter **1**. Choose **Save**.
2. Stop and start the Amazon EC2 instance.
3. After the instance is stopped and restarted, go to the **Alarms** page in the CloudWatch console to confirm that the alarm was created.  You should find a new alarm named **AutoAlarm-<instance id omitted>-StatusCheckFailed-GreaterThanThreshold-1-5m-1p-exampleDescription**.

## Creating a specific alarm for a specific AWS Lambda function using tags

You can create alarms that are specific to an individual AWS Lambda function by adding a tag to the instance using the tag key syntax described in [changing the default alarm set](#changing-the-default-alarm-set).

## Deploying in a multi-region, multi-account environment

You can deploy the CloudWatchAutoAlarms lambda function into a multi-account, multi-region environment by using [CloudFormation StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html).

Follow [steps 1 through 7 in the normal deployment process](#deploy).   For step #3 and step #4, enter your AWS Organizations ID for the **OrganizationID** parameter in the [sample S3 CloudFormation template](./CloudWatchAutoAlarms-S3.yaml) and [sample SNS CloudFormation template](./CloudWatchAutoAlarms-SNS.yaml).  This will update the resource policy to allow access to all accounts in your AWS organization.

Continúe con los siguientes pasos para implementar un AWS StackSet administrado por el servicio para la función lambda CloudWatchAutoAlarms. Esto implementará la función Lambda de CloudWatchAutoAlarms en las unidades organizativas que especifique. La función lambda también se implementará automáticamente en cuentas nuevas en la organización de AWS.

1. Use the [CloudWatchAutoAlarms CloudFormation template](./CloudWatchAutoAlarms.yaml) to deploy the Lambda function across multiple regions and accounts in your AWS Organization.  Este tutorial implementa un CloudFormation StackSet administrado por el servicio en la cuenta master de AWS Organizations. También debe especificar el ID de la cuenta donde se creó el bucket S3 de implementación para que se utilice el mismo bucket S3 en todas las implementaciones de cuentas de su organización. Utilice el siguiente comando para implementar el servicio CloudFormation StackSet administrado:

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

   Once the deployment is complete, the status will change from **RUNNING** to **SUCCEEDED**.

## Security

Consulte [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) para obtener más información.

## License

Esta biblioteca tiene la licencia MIT-0. Para mas, ver el archivo de LICENCIA.