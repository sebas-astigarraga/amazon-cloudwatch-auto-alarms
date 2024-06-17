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

   If you created an S3 bucket using [this sample S3 CloudFormation template](./CloudWatchAutoAlarms-S3.yaml) in step 3, then you can get the bucket name from the AWS Management console or run the following AWS CLI command:

       aws cloudformation describe-stacks --stack-name amazon-cloudwatch-auto-alarms-s3-bucket \
       --query "Stacks[0].Outputs[?ExportName=='amazon-cloudwatch-auto-alarms-bucket-name'].OutputValue" \
       --output text \
       --region <enter your aws region id, e.g. "us-east-1">

8. Deploy the AWS lambda function using the deployment package you uploaded to your S3 bucket:

       aws cloudformation create-stack --stack-name amazon-cloudwatch-auto-alarms \
       --template-body file://CloudWatchAutoAlarms.yaml \
       --capabilities CAPABILITY_IAM \
       --parameters ParameterKey=S3DeploymentKey,ParameterValue=amazon-cloudwatch-auto-alarms.zip \
       ParameterKey=S3DeploymentBucket,ParameterValue=<S3 bucket with your deployment package> \
       ParameterKey=AlarmNotificationARN,ParameterValue=<SNS Topic ARN for Alarm Notifications> \
       --region <enter your aws region id, e.g. "us-east-1">

   If you don't want to enable SNS notifications, you can set the **ParameterValue** to **""** for **AlarmNotificationARN**.

   You can retrieve the SNS Topic ARN from step #3 for the **AlarmNotificationARN** parameter value by running the following command:

       aws cloudformation describe-stacks --stack-name amazon-cloudwatch-auto-alarms-sns-topic \
       --query "Stacks[0].Outputs[?ExportName=='amazon-cloudwatch-auto-alarms-sns-topic-arn'].OutputValue" \
       --output text --region <enter your aws region id, e.g. "us-east-1">

## Activate

### Amazon EC2
In order to create the default alarm set for an Amazon EC2 instance or AWS Lambda function, you simply need to tag the Amazon EC2 instance or AWS Lambda function with the activation tag key defined by the **ALARM_TAG** environment variable.  The default tag activation key is **Create_Auto_Alarms**.

For Amazon EC2 instances, you must add this tag during instance launch or you can add this tag at any time to an instance and then stop and start the instance in order to create the default alarm set as well as any custom, instance specific alarms.

You can also manually invoke the CloudWatchAutoAlarms lambda function with the following event payload to create / update EC2 alarms without having to stop and start your EC2 instances:

```json
{
  "action": "scan"
}
```
You can do this with a test execution of the CloudWatchAUtoAlarms AWS Lambda function.  Open the AWS Lambda Management Console and perform a test invocation from the **Test** tab with the payload provided here.

The [CloudWatchAutoAlarms.yaml](CloudWatchAutoAlarms.yaml) template includes two CloudWatch event rules.  One invokes the Lambda function on `running` and `terminated` instance states.  The other invokes the Lambda function on a daily schedule.  The daily scheduled event will update any existing alarms and also create any alarms with wildcard tags. 

### Amazon RDS

For Amazon RDS, you can add this tag to an RDS database cluster or database instance at any time in order to create the default alarm set as well as any custom alarms that have been specified as tags on the cluster or instance.


### AWS Lambda

For AWS Lambda, you can add this tag to an AWS Lambda function at any time in order to create the default alarm set as well as any custom, function specific alarms.


## Notification Support

You can define an Amazon Simple Notification Service (Amazon SNS) topic that the Lambda function will specify as the notification target for created alarms. The deployment instructions include an SNS topic that you can deploy and use with the solution.  You provide the Amazon SNS Topic Amazon Resource Name (ARN) with the **AlarmNotificationARN** parameter when you deploy the CloudWatchAutoAlarms.yaml CloudFormation template.  This parameter sets the **`DEFAULT_ALARM_SNS_TOPIC_ARN`** environment variable to the ARN you specified.  If you leave the **AlarmNotificationARN** parameter value blank, then this environment variable is not set and created alarms won't use notifications.  

The solution also enables you to specify a unique SNS topic per AWS resource by setting a tag with key **`notify`** and the value set to the SNS topic ARN that should be targeted for alarms for that specific resource.  For any resources that don't have the **`notify`** tag set, the default SNS topic ARN will be used. 

You can apply a tagging strategy that includes the **`notify`** tag for groups of resources to notify on specific groups of resources.  For example, consider a tag with key **`Team`** and value **`Windows`**.  You could align tagging of this specific key / value with the SNS topic for Windows support(e.g. **`notify`**: arn:aws:sns:us-east-1:123456789012:WindowsSupport)

## Changing the default alarm set

You can add, remove, and customize alarms in the default alarm set.  The default alarms are defined in the **default_alarms** python dictionary in [cw_auto_alarms.py](src/cw_auto_alarms.py).

In order to create an alarm, you must uniquely identify the metric that you want to alarm on.  Standard Amazon EC2 metrics include the **InstanceId** dimension to uniquely identify each standard metric associated with an EC2 instance.  If you want to add an alarm based upon a standard EC2 instance metric, then you can use the tag name syntax:
AutoAlarm-AWS/EC2-\<**MetricName**>-\<**ComparisonOperator**>-\<**Period**>-\<**EvaluationPeriods**>-\<**Statistic**>-\<**Description**>
This syntax doesn't include any dimension names because the InstanceId dimension is used for metrics in the **AWS/EC2** namespace.  These AWS provided EC2 metrics are common across all platforms for EC2.

Similarly, AWS Lambda metrics include the **FunctionName** dimension to uniquely identify each standard metric associated with an AWS Lambda function.  If you want to add an alarm based upon a standard AWS Lambda metric, then you can use the tag name syntax:
AutoAlarm-AWS/Lambda-\<**MetricName**>-\<**ComparisonOperator**>-\<**Period**>-\<**EvaluationPeriods**>-\<**Statistic**>-\<**Description**>
You can add any standard Amazon CloudWatch metric for Amazon EC2 or AWS Lambda into the **default_alarms** dictionary under the **AWS/EC2** or **AWS/Lambda** dictionary key using this tag syntax.

## Wildcard support for dimension values on EC2 instance alarms

The solution allows you to specify a wildcard for a dimension value in order to create CloudWatch alarms for all dimension values.  This is particularly useful for creating alarms for all partitions and drives on a system or where the value of a dimension is not known or can vary across EC2 instances.

For example, the CloudWatch agent publishes the `disk_used_percent` metric for disks attached to a Linux EC2 instance.  The dimensions for this metric for Amazon Linux are `device name`, `fstype`, and `path`.

The alarm tag for this metric is hardcoded in the `default_alarms` python dictionary in `cw_auto_alarms.py` to create an alarm for the root volume whose default dimensions and values are: 

* device: nvme0n1p1
* fstype: xfs
* path: /

this is equivalent to the following default tag in the solution: 

```
AutoAlarm-CWAgent-disk_used_percent-device-nvme0n1p1-fstype-xfs-path-/-GreaterThanThreshold-5m-1-Average-Created_by_CloudWatchAutoAlarms
```

If you want to alarm on all disks attached to an EC2 instance then you must specify the device name, file system type, and path dimension values for each disk, which will vary.  Each EC2 instance may also have a different number of disks and different dimension values.

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

If your EC2 instance had two disks with the following dimensions:

*Disk 1*
* device: nvme0n1p1
* fstype: xfs
* path: /

*Disk 2*
* device: nvme1n1p1
* fstype: xfs
* path: /disk2

Then two alarms would be created using a `*` wildcard for the `device` and `path` dimensions:
* AutoAlarm-\<InstanceId>-CWAgent-disk_used_percent-device-nvme0n1p1-fstype-xfs-path-/-GreaterThanThreshold-80-5m-1p-Average-Created_by_CloudWatchAutoAlarms
* AutoAlarm-\<InstanceId>-CWAgent-disk_used_percent-device-nvme1n1p1-fstype-xfs-path-/disk2-GreaterThanThreshold-80-5m-1p-Average-Created_by_CloudWatchAutoAlarms


In order to identify the dimension values, the solution queries CloudWatch metrics to identify all metrics that match the fixed dimension values for the metric name specified.  It then iterates through the dimensions whose values are specified as a wildcard to identify the specific dimension values required for the alarm. 

Because the solution relies on the available metrics in CloudWatch, it will only work after the CloudWatch agent has published and sent metrics to the CloudWatch service.  Since the solution is designed to run on instance launch, these metrics will not be available on first start since the CloudWatch service will not have received them yet.  

In order to resolve this, you should schedule the solution to run on schedule using the `scan` payload:
```json 
{
"action": "scan"
}
```

This will provide sufficient time for the CloudWatch agent to publish metrics for new instances.  You can schedule the frequency of execution based on the acceptable timeframe for which wildcard based alarms for new instances are not yet created.


## Creating CloudWatch Anomaly Detection Alarms

CloudWatch Anomaly Detection Alarms are supported using the comparison operators `LessThanLowerOrGreaterThanUpperThreshold`, `LessThanLowerThreshold`, or `GreaterThanUpperThreshold`.

When you specify one of these comparison operators, the solution creates an anomaly detection alarm and uses the value for the tag key as the threshold.  Refer to the [CloudWatch documentation for more details on the threshold and anomaly detection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html).

CloudWatch Anomaly detection uses machine learning models based on the metric, dimensions, and statistic chosen.  If you create an alarm without a current model, CloudWatch Alarms creates a new model using these parameters from your alarm configuration.  
For new models, it can take up to 3 hours for the actual anomaly detection band to appear in your graph. It can take up to two weeks for the new model to train, so the anomaly detection band shows more accurate expected values.  Refer to the documentation for more details.

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

You can uncomment and update this code to test out anomaly detection support.  

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

Continue with the following steps to deploy a service managed AWS StackSet for the CloudWatchAutoAlarms lambda function.  This will deploy the CloudWatchAutoAlarms Lambda function into the organization units that you specify.  The lambda function will also be automatically deployed to new accounts in the AWS organization.

1. Use the [CloudWatchAutoAlarms CloudFormation template](./CloudWatchAutoAlarms.yaml) to deploy the Lambda function across multiple regions and accounts in your AWS Organization.  This walkthrough deploys a service managed CloudFormation StackSet in the AWS Organizations master account.  You must also specify the account ID where the S3 deployment bucket was created so the same S3 bucket is used across account deployments in your organization.  Use the following command to deploy the service managed CloudFormation StackSet:

       aws cloudformation create-stack-set --stack-set-name amazon-cloudwatch-auto-alarms \
       --template-body file://CloudWatchAutoAlarms.yaml \
       --capabilities CAPABILITY_NAMED_IAM \
       --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false \
       --permission-model SERVICE_MANAGED \
       --parameters ParameterKey=S3DeploymentKey,ParameterValue=amazon-cloudwatch-auto-alarms.zip \
       ParameterKey=S3DeploymentBucket,ParameterValue=<S3 bucket with your deployment package> \
       --region <enter your aws region id, e.g. "us-east-1">

      2. After the StackSet is created, you can specify which AWS accounts and regions the StackSet should be deployed.  For service managed StackSets, you specify your AWS Organization ID or AWS Organizational Unit IDs to deploy the lambda function to all current and future accounts within them.   Use the following AWS CLI command to deploy the StackSet to your organization / organizational units:

       aws cloudformation create-stack-instances --stack-set-name amazon-cloudwatch-auto-alarms \
       --operation-id amazon-cloudwatch-auto-alarms-deployment-$(date | md5) \
       --deployment-targets OrganizationalUnitIds=<Enter the target OUs where the lambda function should be deployed> \
       --regions <enter the target regions where the lambda function should be deployed e.g. "us-east-1"> \
       --region <enter your aws region id, e.g. "us-east-1">

   You can monitor the progress and status of the StackSet operation in the AWS CloudFormation service console.

   Once the deployment is complete, the status will change from **RUNNING** to **SUCCEEDED**.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.