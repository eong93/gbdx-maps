# This example details how to run OpenSkyNet (OSN) to return aircraft detections.
*Requires gbdxtools Python module*
[Instructions on how to install gbdxtools](http://gbdxtools.readthedocs.io/en/latest/user_guide.html)

```python
from gbdxtools import Interface
gbdx = Interface()
```
Import/define tools from module

```python
wkt_string = "POLYGON((-77.49189376831055 38.97302269384043,-77.43335723876953 38.97302269384043,-77.43335723876953 38.920688310253,-77.49189376831055 38.920688310253,-77.49189376831055 38.97302269384043))"
```
Create a Well Known Text string to use as a bounding box to search the catalog with. (This bounding box is over Dulles International Airport). 

```python
filters = ["sensorPlatformName = 'WORLDVIEW02'", "cloudCover < 15"]
```
Create additional filters to narrow down catalog results.

```python
results = gbdx.catalog.search(searchAreaWkt=wkt_string, startDate="2014-01-01T00:00:00.000Z", endDate="2014-12-31T00:00:00.000Z", filters=filters)

print results
```
Query the catalog with created bounding box, filters, and date requirements.

```python
order_id = gbdx.ordering.order('103001003A230A00')
print order_id
gbdx.ordering.status(order_id)
```
From the catalog search results pick a Catalog ID to run OSN on. For this example, Catalog ID `103001003A230A00` was picked and was ordered. Once a Catalog ID is ordered, the s3 location of the image will be returned.

```python
data = "s3://receiving-dgcs-tdgplatform-com/055378720010_01_003"
```
Set image location as a varaible to be used in tasks.

```python
aoptask = gbdx.Task("AOP_Strip_Processor", data=data, enable_dra=True, enable_pansharpen=True, enable_acomp=True, ortho_epsg='UTM', bands='PAN+MS', ortho_pixel_size='0.5', ortho_interpolation_type='Bilinear')
```
OSN requires images to be processed with the `AOP_Strip_Processor` GBDX task with the appropriate parameters.

```python
croptask = gbdx.Task("CropGeotiff", data=aoptask.outputs.data, output_to_root_dir=True, wkt="POLYGON((-77.49189376831055 38.97302269384043,-77.43335723876953 38.97302269384043,-77.43335723876953 38.920688310253,-77.49189376831055 38.920688310253,-77.49189376831055 38.97302269384043))")
```
For this example, the AOP'd image will be cropped by the above bounding box. (OSN may timeout due to large images)

```python
osntask = gbdx.Task("openskynet-v5:0.0.2", data=croptask.outputs.data, model='s3://vector-lulc-models/0ad86e8caf6d9000.zip', log_level='trace', confidence='0.85', pyramid=True, pyramid_window_sizes='[150, 80]', pyramid_step_sizes='[40, 20]', step_size='15', tags='Airliner, Fighter, Helicopter')
```
The actual OSN GBDX task with appropriate parameters.`s3://vector-lulc-models/0ad86e8caf6d9000.zip` is the location of the OSN model for this example.

```python
workflow = gbdx.Workflow([aoptask, croptask, osntask])
workflow.savedata(osntask.outputs.results, "some_folder_under_account_prefix")
workflow.execute()
```
This workflow will run the above tasks in the specified order. Save the outputs of OSN to desired folder on s3.

[In-depth documentation on gbdxtools](http://gbdxtools.readthedocs.io/en/latest/running_workflows.html)
