# product-terraform-module

Ensure the API server is up and running.
https://github.com/vpovarna/go-mux-api

To provision products on the product API serer, a provider needs to exist on: `~/.terraform.d/plugins/`

Link: https://github.com/vpovarna/terraform-provider-product-api

## Intialize provider
Add the provider block to terraform.tf

```
provider "product" {
  version = "1.0.0"
  address = "http://localhost"
  port    = "18010"
}
```
Run `terrafom init` to initialize the provider

## Add terraform version constrains inside the module
Add teh following block insinde the terraform.tf file.

```
terraform {
  required_version = ">= 0.12"
}
```

Run `terraform plan`

## Create a resource
Add this resource block to `main.tf`
```
resource "product" "test" {
  pid = 1
  name = "TestProduct"
  price = 11.00
}
```

Run `terraform plan` and check the output. 
In order to create the product, run: `terraform apply`

You can see the product created by accessing the api endpoint: `http://localhost:18010/products`

## Get the resource output
Run `terrafom output test`. You can see the created product information. 

## Delete the created product
Run `terraform destroy`. You can see that there are no more products on the server by accessing: `http://localhost:18010/products`

## Create multiple products using count
Add this resource block to `main.tf`