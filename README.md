# ca-dmv-autonomous-vehicle-accidents
Mirror of CA.gov's "Report of Traffic Accident Involving an Autonomous Vehicle (OL 316)"


Mirror page:

https://wgetsnaps.github.io/ca-dmv-autonomous-vehicle-accidents

Original page:

https://www.dmv.ca.gov/portal/dmv/detail/vr/autonomous/autonomousveh_ol316


## wget code

```sh

wget --recursive --level 1 \
  --adjust-extension \
  --page-requisites \
  --convert-links \
  --no-directories \
  --output-file /dev/stdout \
  https://www.dmv.ca.gov/portal/dmv/detail/vr/autonomous/autonomousveh_ol316 \
  | tee ./wget.log
```
