```python
from gbdxtools import Interface
```

gbdx = Interface()

wkt_string = "POLYGON((-77.49189376831055 38.97302269384043,-77.43335723876953 38.97302269384043,-77.43335723876953 38.920688310253,-77.49189376831055 38.920688310253,-77.49189376831055 38.97302269384043))"
filters = ["sensorPlatformName = 'WORLDVIEW02'", "cloudCover < 15"]
results = gbdx.catalog.search(searchAreaWkt=wkt_string, startDate="2014-01-01T00:00:00.000Z", endDate="2014-12-31T00:00:00.000Z", filters=filters)
print results

order_id = gbdx.ordering.order('103001003A230A00')
print order_id
gbdx.ordering.status(order_id)

data = "s3://receiving-dgcs-tdgplatform-com/055378720010_01_003"
aoptask = gbdx.Task("AOP_Strip_Processor", data=data, enable_dra=True, enable_pansharpen=True, enable_acomp=True, ortho_epsg='UTM', bands='PAN+MS', ortho_pixel_size='0.5', ortho_interpolation_type='Bilinear')
croptask = gbdx.Task("CropGeotiff", data=aoptask.outputs.data, output_to_root_dir=True, wkt="POLYGON((-77.49189376831055 38.97302269384043,-77.43335723876953 38.97302269384043,-77.43335723876953 38.920688310253,-77.49189376831055 38.920688310253,-77.49189376831055 38.97302269384043))")
osntask = gbdx.Task("openskynet-v5:0.0.2", data=croptask.outputs.data, model='s3://vector-lulc-models/0ad86e8caf6d9000.zip', log_level='trace', confidence='0.85', pyramid=True, pyramid_window_sizes='[150, 80]', pyramid_step_sizes='[40, 20]', step_size='15', tags='Airliner, Fighter, Helicopter')
workflow = gbdx.Workflow([aoptask, croptask, osntask])
workflow.savedata(osntask.outputs.results, "eong/osn2")
workflow.execute()
