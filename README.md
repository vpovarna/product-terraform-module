# product-terraform-module

Ensure the API server is up and running.
https://github.com/vpovarna/go-mux-api

To provision products on the product API serer, a provider needs to exist on: `~/.terraform.d/plugins/`

Link: https://github.com/vpovarna/terraform-provider-product-api

## Check the installed terraform version
```
terraform --version
```

## Initialize provider
Add the provider block to terraform.tf

```
provider "product" {
  version = "1.0.0"
  address = "http://localhost"
  port    = "18010"
}
```
Run `terraform init` to initialize the provider

## Add terraform version constrains inside the module
Add the following block inside the terraform.tf file.

```
terraform {
  required_version = ">= 0.12"
}
```

Run `terraform plan`. 
```
------------------------------------------------------------------------

No changes. Infrastructure is up-to-date.

This means that Terraform did not detect any differences between your
configuration and real physical resources that exist. As a result, no
actions need to be performed.

```

Run `terraform apply`.
```
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

## Create a resource
Add this resource block to `main.tf`

product resources: https://github.com/vpovarna/terraform-provider-product-api/blob/main/provider/resource_product.go 

```
resource "product" "test" {
  pid   = 1
  name  = "TestProduct"
  price = 11.00
}
```

Run `terraform plan` and check the output. 
```
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # product.test will be created
  + resource "product" "test" {
      + id    = (known after apply)
      + name  = "TestProduct"
      + pid   = 1
      + price = 11
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```
In order to create the product, run: `terraform apply`. Once you double check the output, enter the value: `yes`.
If you don't want to enter `yes` every time you run `terraform apply` you can use `--auto-approve` flag
Output:

```
product.test: Creating...
product.test: Creation complete after 0s [id=1]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

```

You can see the product created by accessing the api endpoint: `http://localhost:18010/products`

## Get the resource output
Add the following block inside the `main.tf` file:
```
output "testProduct" {
    value = "Test Value"
}
```
Run `terraform apply --auto-approve`
We should see this output in the terminal:
```
Outputs:

testProduct = Test Value
```

Get the price of the product we just created. Add the following block into the `main.tf` file:
```
output "testProductPrice" {
    value = product.test.price
}
```
Run `terraform apply --auto-approve`
```
$ terraform apply --auto-approve
product.test: Refreshing state... [id=1]

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

testProduct = Test Value
testProductPrice = 11
```

If we just want to see the output values, we can run: `terraform output`:
```
$ terraform output
testProduct = Test Value
testProductPrice = 11
```
If we want just the output of the testProductPrice, run: `terraform output testProductPrice`:
```
$ terraform output testProductPrice
11
```

Add: `sensitive = true` to the output block and run the same command again: `terraform apply --auto-approve`
The output of the testProductPrice will not be display in the terminal, but it will be in the state file. 
```
Outputs:

testProduct = Test Value
testProductPrice = <sensitive>
```

As a best practice, the `output` blocks should stay into the `outputs.tf` file.
After you move them to the `outputs.tf`, run: `terraform apply --auto-approve`
You should get the same results:
```
Outputs:

testProduct = Test Value
testProductPrice = <sensitive>
```

## Declaring variables
Let's create a variable inside the `main.tf`
Variables type: https://www.terraform.io/docs/language/expressions/types.html#types 

```
variable "newPrice" {
  description = "New product price"  
  type        = number
  default     = 10.00 
}
```

We can use this variable inside a new product resource. Ex:
```
resource "product" "test2" {
  pid   = 2
  name  = "SecondTestProduct"
  price = var.newPrice
}
```

Run `terraform plan` and if everything it's correct, run: `terraform apply --auto-approve`
As a best practice, the `variables` blocks should stay into the `variable.tf` file.

To overwrite the default value we can: 
- Remove the default value, and then run: `terraform apply`
It will prompt for an input value. 
```
$ terraform apply
var.newPrice
  Enter a value:
```
The test2 product will be updated in place, and will be ask for confirmation.
```
  # product.test2 will be updated in-place
  ~ resource "product" "test2" {
        id    = "2"
        name  = "SecondTestProduct"
        pid   = 2
      ~ price = 10 -> 1
    }
```
- Add variables value to a `terraform.tfvars` file.
Create a new file: `terraform.tfvars` inside the module, and add the: `newPrice = 2.00` variable to it. Terraform will automatically loading the file for you. 
Run: `terraform apply`
The value will be loaded and the `test2` product will be updated:
```
  # product.test2 will be updated in-place
  ~ resource "product" "test2" {
        id    = "2"
        name  = "SecondTestProduct"
        pid   = 2
      ~ price = 1 -> 2
    }
```

Create a local variable.
In the `main.tf` file create a new `locals` block:
```
locals {
  newPrice = 3.00
}
```

You can reference the value inside a new resource. Ex:
```
resource "product" "test3" {
  pid   = 3
  name  = "ThirdTestProduct"
  price = local.newPrice
}
```

Run: `terraform apply`. You should see the following output:
```
Terraform will perform the following actions:

  # product.test3 will be created
  + resource "product" "test3" {
      + id    = (known after apply)
      + name  = "ThirdTestProduct"
      + pid   = 3
      + price = 3
    }
```

Rename the `terraform.tfvars` to a custom name. Ex: `products.tfvars` file. To load the renamed file, run:
`terraform apply -var-file products.tfvars`. This is useful for multiple environment dev/stage/prod tfvars files. 

- Passing variables through the command line:
`terraform apply -var="newPrice=4.00"`
Output will be:
```
  # product.test2 will be updated in-place
  ~ resource "product" "test2" {
        id    = "2"
        name  = "SecondTestProduct"
        pid   = 2
      ~ price = 2 -> 4
    }

```

- Set variables through environment variables. 
Terraform will look for `TF_VAR` env variable prefix. Ex: `export TF_VAR_newPrice=5.00`

## Create multiple products using count meta argument

In the `main.tf` file add the following block:

```
resource "product" "test4" {
  pid   = 100 + count.index
  name  = "${var.name}-${count.index}"
  price = var.price
  count = var.productCount
}
```

The resource requires a couple of variables. So let's add them to the `variables.tf` file:
```
variable "productCount" {
  description = "Number of test products"
  type        = number
  default     = 4
}

variable "price" {
    description = "default price"
    type        = string
    default     = 10.00
}

variable "name" {
  description = "Product Name"
  type        = string
  default     = "demo"
}
```

Run: `terraform plan`. Output should be:
```
Terraform will perform the following actions:
  # product.test4[0] will be created
  + resource "product" "test4" {
      + id    = (known after apply)
      + name  = "demo-0"
      + pid   = 4
      + price = 10
    }

  # product.test4[1] will be created
  + resource "product" "test4" {
      + id    = (known after apply)
      + name  = "demo-1"
      + pid   = 4
      + price = 10
    }

  # product.test4[2] will be created
  + resource "product" "test4" {
      + id    = (known after apply)
      + name  = "demo-2"
      + pid   = 4
      + price = 10
    }

  # product.test4[3] will be created
  + resource "product" "test4" {
      + id    = (known after apply)
      + name  = "demo-3"
      + pid   = 4
      + price = 10
    }

Plan: 4 to add, 0 to change, 0 to destroy.
```

## If conditions:
Add the following block to the `main.tf` file:
```
resource "product" "test5" {
  pid   = 6
  name  = "LastTest"
  price = var.higherPrice ? 100.00 : 5
}
```
In `variables.tf` add the following variable definition:
```
variable "higherPrice" {
    description = "price condition"
    type        = bool
    default     = true
}
```

Run `terraform plan` and the output should be:
```
  # product.test6 will be created
  + resource "product" "test6" {
      + id    = (known after apply)
      + name  = "LastTest"
      + pid   = 6
      + price = 100
    }
```

## Create multiple resources using for_each

Let's define a new variable of type map in `variables.tf` file:
```
variable "custom_products" {
  description = "Product maps"
  type        = map(string)
  default = {
    Iphone = 10.00
    IPad = 20.00
  }
}
```

To use this variable, add the following block in `main.tf`:
```
resource "product" "test6" {
  for_each = var.custom_products

  pid   = 7
  name  = each.key
  price = each.value
}
```

Run `terraform plan`. The output should be:
```
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # product.test6["IPad"] will be created
  + resource "product" "test6" {
      + id    = (known after apply)
      + name  = "IPad"
      + pid   = 7
      + price = 20
    }

  # product.test6["Iphone"] will be created
  + resource "product" "test6" {
      + id    = (known after apply)
      + name  = "Iphone"
      + pid   = 7
      + price = 10
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```
Both products have the same pid value, so we need a different structure.

We should be able to fix this, using an object structure. 
Let's modify the `custom_products` variable from `variable.tf` file with:
```
variable "custom_products" {
  description = "Product maps"
  type        = map(object({
      pid   = number
      name  = string
      price = number
  }))
  default = {
      iphone = {
          pid   = 21
          name  = "Iphone"
          price = 20.00
      },
      ipad = {
          pid   = 22
          name  = "Ipad"
          price = 100.00
      }
  }
}
```

The product `test6` needs to be modified as well:
```
resource "product" "test6" {
  for_each = var.custom_products

  pid   = each.value.pid
  name  = each.value.name
  price = each.value.price
}
```

Run: `terraform.plan`. The output should be:
```
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # product.test6["ipad"] will be created
  + resource "product" "test6" {
      + id    = (known after apply)
      + name  = "Ipad"
      + pid   = 22
      + price = 100
    }

  # product.test6["iphone"] will be created
  + resource "product" "test6" {
      + id    = (known after apply)
      + name  = "Iphone"
      + pid   = 21
      + price = 20
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```

## Remove a resource from the state
To delete a specific resource, run: `terraform state rm 'product.test3'`
Output: 
`$ terraform state rm 'product.test3'
Removed product.test3
Successfully removed 1 resource instance(s).`

Items removed from the Terraform state are not physically destroyed. Items removed from the Terraform state are only no longer managed by Terraform. 

`test3` is still available on the server: `http://localhost:18010/product/3`

## Destroying products

Delete a specific resource type name: `terraform destroy -target product.test6 `
Output:
```
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # product.test6["ipad"] will be destroyed
  - resource "product" "test6" {
      - id    = "22" -> null
      - name  = "Ipad" -> null
      - pid   = 22 -> null
      - price = 100 -> null
    }

  # product.test6["iphone"] will be destroyed
  - resource "product" "test6" {
      - id    = "21" -> null
      - name  = "Iphone" -> null
      - pid   = 21 -> null
      - price = 20 -> null
    }

Plan: 0 to add, 0 to change, 2 to destroy.
```

To delete all the created resources, run `terraform destroy`. You can see that there are no more products on the server by accessing: `http://localhost:18010/products`
