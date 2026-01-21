# Terraform tricks and tips

## `Locals{}` And Loops

### The `locals {}` Block

Locals are internal variables used to simplify complex expressions or avoid repetition within a module. Unlike input variables, they cannot be set from outside the module

```hcl
locals {
  # Hardcoded values
  environment = "prod"
  
  # Transformations using functions
  app_prefix = upper("app-${local.environment}")
  
  # Complex data structures
  user_map = {
    alice = { role = "admin", dept = "it" }
    bob   = { role = "user",  dept = "sales" }
  }
}
```

### The `for_each` Meta-Argument

`for_each` is used to create multiple instances of a resource or module. It requires a map or a set of strings. 
- **`each.key`**: The map key (or set member) for the current instance.
- **`each.value`**: The value associated with that key.  

```hcl
locals {
  user_map = {
    alice = { role = "admin", dept = "it" }
    bob   = { role = "user",  dept = "sales" }
  }
}

resource "aws_iam_user" "example" {
  # Iterating over the user_map defined in locals
  for_each = local.user_map

  name = each.key # "alice", "bob"
  tags = {
    Department = each.value.dept # "it", "sales"
    Role       = each.value.role
  }
}
```

### The `for` Expression

While `for_each` creates resources, the `for` expression transforms collections into new ones. It is often used inside a locals block to prepare data for a `for_each` loop. 
- **`List to List`**: `[for item in list : transform(item)]`
- **`Map to Map`**: `{for k, v in map : k => transform(v)}`
- **`Filtering`**: Use `if` at the end to filter elements.  

```hcl
locals {
  # 1. Transform a list into a list of uppercase names
  raw_names = ["web", "api", "db"]
  upper_names = [for n in local.raw_names : upper(n)]

  # 2. Filter a map (create a map of only admins)
  admins = {
    for name, details in local.user_map : name => details
    if details.role == "admin"
  }
}
```

### Summarize

| Feature | Purpose | Input Type |
|---------|---------|------------|
| `locals {}` | Internal variable storage & calculation. | Any HCL expression. |
| `for_each` | Creating multiple resources. | Map or Set of strings. |
| `for` loop | Transforming or filtering data. | List, Map, Set, or Tuple. |


```hcl
variable "server_names" {
  type    = list(string)
  default = ["web-01", "web-02"]
}

locals {
  # Prepare a map for for_each (must be a map or set)
  server_config = {
    for name in var.server_names : name => {
      instance_type = "t3.micro"
      tag_name      = "Server-${name}"
    }
  }
}

resource "aws_instance" "app" {
  for_each = local.server_config

  instance_type = each.value.instance_type
  tags = {
    Name = each.value.tag_name
  }
}
```

## Variable types

| Type | Description | Ordering | Special Rules | Use Case | Syntax |
|------|-------------|----------|---------------|----------|--------|
| **List** | An ordered sequence of values that must all be of the same type. | Preserved (access elements via index [0], [1]) | Duplicates: Allowed | When the order of resources matters, such as a sequence of network subnets | `["us-east-1a", "us-east-1b"]` |
| **Map** | A collection of key-value pairs where all values must be of the same type. | Not guaranteed; maps are unordered lookups | Keys: Must be unique strings | Lookup tables, such as mapping environment names to instance sizes (e.g., dev = "t3.micro") | `{ env = "prod", region = "us-west-1" }` |
| **Set** | An unordered collection of unique values of the same type. | Discarded (no index access) | Duplicates: Automatically removed | When you need a unique list of items where order is irrelevant, like a list of distinct IP addresses or security group IDs | Created using the `toset()` function on a list |
| **Tuple** | An ordered sequence of values where each position can have a different type. | Preserved (access via index) | The number of elements and their specific types are part of the tuple's schema | Returning a fixed set of mixed data from a module, like ["vpc-123", 80, true] | `["web-server", 5, false]` |  



| Type | Ordered? | Duplicate Values? | Element Types | Common Use |
|------|----------|-------------------|---------------|------------|
| List | Yes | Yes | Same | Sequential resources |
| Map | No | No (Unique keys) | Same | Configuration lookups |
| Set | No | No (Unique values) | Same | Unique tag/ID lists |
| Tuple | Yes | Yes | Mixed | Fixed positional data |

### Examples Snippets

#### List (Ordered, Homogeneous)

Lists are best when the order of elements matters, such as selecting a specific subnet index.  

```hcl
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# Accessing by index
resource "aws_subnet" "example" {
  availability_zone = var.availability_zones[0] # Returns "us-east-1a"
  # ... other config
}
```

#### Map (Unordered, Unique Keys)
Maps are ideal for lookup tables where you want to fetch a value based on a specific label, like environment-specific instance sizes.

```hcl
variable "instance_sizes" {
  type = map(string)
  default = {
    dev  = "t3.micro"
    prod = "m5.large"
  }
}

# Accessing by key
resource "aws_instance" "app" {
  instance_type = var.instance_sizes["dev"] # Returns "t3.micro"
  # ... other config
}
```

#### Set (Unordered, Unique Values)
Sets are used when you need to ensure every item is unique. They are commonly used with for_each to create resources from a unique list of names. 

```hcl
locals {
  # toset() removes duplicates: ["user1", "user2"]
  unique_users = toset(["user1", "user2", "user1"])
}

resource "aws_iam_user" "users" {
  for_each = local.unique_users
  name     = each.key
}
```

#### Tuple (Ordered, Mixed Types)
Tuples allow you to store multiple different data types in a specific sequence. They are often used for fixed-position configurations. 

```hcl
variable "db_config" {
  # index 0: Name (string), index 1: Port (number), index 2: Is Public (bool)
  type    = tuple([string, number, bool])
  default = ["inventory-db", 5432, false]
}

output "db_port" {
  value = var.db_config[1] # Returns 5432
}
```

#### Complex example

A classic complex example is creating multiple Security Group Rules from a single configuration object where each rule might have multiple ports.  

```hcl
variable "security_config" {
  type = map(object({
    description = string
    ports       = list(number)
    cidr_blocks = list(string)
  }))
  default = {
    "web" = {
      description = "Web Traffic"
      ports       = [80, 443]
      cidr_blocks = ["0.0.0.0/0"]
    }
    "db" = {
      description = "Database Access"
      ports       = [5432]
      cidr_blocks = ["10.0.0.0/16"]
    }
  }
}

locals {
  # We must flatten the map because for_each cannot loop over nested lists directly.
  # This creates a unique key for every "service-port" combination.
  flattened_rules = merge([
    for service_key, config in var.security_config : {
      for port in config.ports : "${service_key}-${port}" => {
        service_name = service_key
        port         = port
        cidr_blocks  = config.cidr_blocks
        description  = config.description
      }
    }
  ]...) # The "..." is the expansion symbol to merge the list of maps into one map
}

resource "aws_vpc_security_group_ingress_rule" "rules" {
  # Best practice in 2026 is using individual rule resources
  for_each = local.flattened_rules

  security_group_id = aws_security_group.main.id
  description       = "${each.value.description} (Port ${each.value.port})"
  
  from_port   = each.value.port
  to_port     = each.value.port
  ip_protocol = "tcp"
  cidr_ipv4   = each.value.cidr_blocks[0] # Simplification for example
}

# Accessing the specific 'web-80' ingress rule
output "web_rule_id" {
  value = aws_vpc_security_group_ingress_rule.rules["web-80"].id
}

# Returns a list of all security group rule IDs
output "all_rule_ids" {
  value = [for r in aws_vpc_security_group_ingress_rule.rules : r.id]
}

# Alternative shorthand using values()
output "all_rule_ids_shorthand" {
  value = values(aws_vpc_security_group_ingress_rule.rules)[*].id
}

# Returns: { "web-80" = "sgr-123", "web-443" = "sgr-456", ... }
output "rules_map" {
  value = { for k, r in aws_vpc_security_group_ingress_rule.rules : k => r.id }
}

resource "example_resource" "next_step" {
  # This creates one resource for every rule created in the previous step
  for_each = aws_vpc_security_group_ingress_rule.rules

  rule_id = each.value.id
  name    = "Copy-of-${each.key}"
}
```

**Summary of References**  
| Target | Syntax |
|--------|--------|
| Specific Instance | `resource_type.name["key"]` |
| All Instances (List) | `[for r in resource_type.name : r.attribute]` |
| All Instances (Map) | `{for k, r in resource_type.name : k => r.attribute}` |


## Built-in Function

### Documentation Overview
The [documentation](https://developer.hashicorp.com/terraform/language/functions) categorizes functions by their purpose, providing detailed usage examples for each:  

| Category | Description | Functions |
|----------|-------------|-----------|
| Numeric Functions | Operations | `abs`, `ceil`, `floor`, `max`, `min` |
| String Functions | Manipulation tools | `split`, `join`, `replace`, `trim`, `upper`, `lower` |
| Collection Functions | Tools | `length`, `merge`, `lookup`, `flatten`, `element` |
| Filesystem Functions | For reading or processing local files | `file`, `templatefile`, `abspath` |
| IP Network Functions | Calculations for CIDR blocks | `cidrsubnet`, `cidrhost` |
| Encoding & Crypto Functions | Handling data formats and security | `jsonencode`, `base64decode`, `sha256` |
| Type Conversion | Converting between types | `tolist`, `tomap`, `toset`, `tostring` |  

To experiment with any function and see its output in real-time using the **Terraform Console**:  
1. Open terminal in a directory with a Terraform configuration.
2. Run the command: `terraform console`.
3. Type a function call to see the result immediately (e.g., `> lower("HELLO")` returns `"hello"`). 

## Resource Dependencies

### Implicit Dependencies 

This is the most common and "Terraform-native" way to handle dependencies. You create an implicit dependency by referencing an attribute of one resource inside the configuration of another.  
Terraform automatically analyzes these references to build a dependency graph.  

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "frontend" {
  # Implicit dependency: This subnet depends on the VPC
  # Terraform will always create the VPC before this subnet
  vpc_id = aws_vpc.main.id 
  
  cidr_block = "10.0.1.0/24"
}
```

### Explicit Dependencies (`depends_on`)
Sometimes a resource depends on another resource behaviorally, but they don't share data (no attribute reference). In these cases, you use the depends_on meta-argument.  
- **Syntax**: It accepts a list of resources or modules.
- **Placement**: It can be used in any resource, data, or module block.  

```hcl
resource "aws_iam_role" "example" {
  name = "example-role"
  assume_role_policy = "..."
}

resource "aws_iam_role_policy" "example" {
  role   = aws_iam_role.example.name
  policy = "..."
}

resource "aws_instance" "example" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"

  # Explicit dependency: The instance needs the IAM policy to be 
  # fully attached/active before it starts, even though the 
  # instance resource doesn't directly reference the policy.
  depends_on = [
    aws_iam_role_policy.example
  ]
}
```

### Dependencies with `for_each`

When using `for_each`, you can still use both methods. If you use `depends_on` with a resource that has `for_each`, the dependent resource will wait for all instances of that `for_each` resource to complete.  

```hcl
resource "aws_sqs_queue" "queues" {
  for_each = toset(["orders", "billing"])
  name     = each.key
}

resource "aws_lambda_function" "processor" {
  # ... lambda config ...

  # This Lambda will wait until BOTH 'orders' and 'billing' queues are created
  depends_on = [aws_sqs_queue.queues]
}
```
### Best Practices

1. **Prefer Implicit**: Only use `depends_on` as a last resort. Implicit dependencies make your code easier to read and maintain.
2. **Module Dependencies**: You can make an entire module depend on another resource/module using `depends_on` at the module call level.
3. **Data Sources**: If a data source depends on a resource being created first, use `depends_on` within the data block to ensure it doesn't try to read information that doesn't exist yet.  

