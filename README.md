## Guide to generate the rfc

- mmark will generate an xml
- the xml can be transformed in rfc with xml2rfc tool
- there is a github action triggered each time there is a release (not tested)

## How to exploit the Github action in this repo

- Edit only the `dev` branch, do not touch the `master` one
- When you want to put the changes in prod, update the version in `package.json` and merge from dev to master
- A new release with the specs will be created