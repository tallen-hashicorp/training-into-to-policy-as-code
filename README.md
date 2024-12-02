# Training Introduction to Policy-as-Code with Sentinel

## What is Sentinel?

Sentinel is a **Policy-as-Code** framework that enables fine-grained, logic-based policies within HashiCorp's ecosystem. It is to policy frameworks what Terraform is to Infrastructure-as-Code. Sentinel features its own language, embedded in HashiCorp products, and integrates seamlessly into provisioning workflows, ensuring policies are enforced during every run. By using **imports**, Sentinel can access data from Terraform plans, configurations, runs, and states, allowing it to enforce detailed policies across your infrastructure.

---

## Benefits of Policy-as-Code

Policy-as-Code offers numerous benefits, particularly in environments where consistency and compliance are critical. By encoding best practices and compliance requirements as code, organizations can enforce standards across teams and projects while reducing the risk of manual errors and misconfigurations. Security best practices are automatically applied from the start, minimizing vulnerabilities and simplifying audits. 

Version control ensures transparency by tracking changes to policies over time, while the ability to detect violations early in the development lifecycle helps to maintain quality. Policies can be reused across multiple projects and environments, promoting efficiency and simplifying maintenance. These benefits allow organizations to treat policy management as a disciplined software development process.

---

## How Organizations Use Sentinel with Terraform

Organizations leverage Sentinel in Terraform to enforce security, control costs, and standardize deployments. For example, policies can mandate that all S3 buckets use a private ACL and encryption via KMS, restrict the roles a cloud provider can assume, or block the use of specific resources, data sources, or provisioners. Cost control policies may limit the sizes of VMs or Kubernetes clusters and enforce monthly spending caps per workspace. 

Sentinel also ensures infrastructure is consistent with organizational standards by requiring resources to include mandatory tags or enforcing the use of modules from a private module registry (PMR). By automating these policies, organizations can efficiently manage compliance, security, and cost-effectiveness.

---

## Writing Sentinel Policies

Sentinel policies define rules enforced during Terraform runs and are written in the **Sentinel Policy Language**. Policies are grouped into **policy sets**, and extensive testing ensures their reliability before enforcement. Testing involves creating mock data to simulate real-world scenarios and adjusting policies as needed.

Policies can be enforced at three levels. **Advisory** enforcement logs violations without blocking runs. **Soft Mandatory** enforcement allows authorized users to override violations, while **Hard Mandatory** enforcement strictly blocks runs with no exceptions. Best practices recommend starting new policies as **Advisory** to gather feedback and confidence before moving to stricter enforcement levels.

---

## Example Policy: Enforcing PMR Usage

This example ensures all Terraform modules come from a private module registry. If any module violates the policy, detailed violation messages are logged.

```sentinel
# Address of the TFC or TFE server
param address default "app.terraform.io"
# Allowed organizations for modules
param organizations

# Identify modules not in the desired PMR
violatingMCs = filter tfconfig.module_calls as index, mc {
 mc.module_address is "" and
 not any organizations as organization {
   strings.has_prefix(mc.source, address + "/" + organization)
 }
}

# Log violations if any
if length(violatingMCs) > 0 and not tfrun.is_destroy {
 print("All modules must come from a private module registry in these organizations:", organizations, " on server", address)
 for violatingMCs as address, mc {
   print("Module", mc.name, "from source", mc.source, "violates policy.")
 }
}
```

---

## Sentinel Testing with Mocks

Testing is a vital step to ensure policies behave as expected. Sentinel tests run via the `sentinel test` command and use mock data to simulate Terraform plans, configurations, and other inputs. These tests allow you to validate both passing and failing conditions, ensuring the policy logic handles real-world scenarios accurately. Mocks provide a way to create test data without relying on live infrastructure, helping to maintain consistency during policy development.

---

## Sentinel Language Basics

Sentinel policies are written as `.sentinel` files and must include a `main` rule to define whether a policy passes or fails. These UTF-8 encoded files execute sequentially, and policies pass only if the `main` rule evaluates to `true`.

For example, a policy to check monthly cost limits might look like this:

```sentinel
# Main rule ensures monthly cost is below threshold
main = rule {
   delta_monthly_cost.less_than(10)
}
```

The `main` rule can incorporate logic from other rules, enabling complex decision-making while keeping the main policy straightforward.

---

## Functions in Sentinel

Functions allow reusable logic within Sentinel policies. Declared using the `func` keyword, functions can accept parameters, perform computations, and return values. For example:

```sentinel
find_resources = func(resource_type) {
 # Perform operations with the parameter
 return true
}
```

Functions simplify policies by encapsulating repetitive logic. A function call might look like this: `s3_buckets = find_resources("aws_s3_bucket")`.

---

## Common Imports in Sentinel

Sentinel integrates deeply with Terraform through **imports**, providing access to plans, configurations, states, and run metadata.

### Using `tfplan/v2`
The `tfplan` import validates attributes in the current Terraform plan. For instance, you can enforce instance types allowed by your organization:

```sentinel
import "tfplan/v2" as tfplan

# Enforce allowed instance types
main = rule {
   (instance_type_allowed) else true
}
```

### Using `tfconfig/v2`
The `tfconfig` import validates configurations such as providers, modules, and variables. 

```sentinel
import "tfconfig"

# Check for AWS provider in configuration
main = rule {
 "aws" in tfconfig.providers
}
```

### Using `tfstate/v2`
The `tfstate` import ensures previously provisioned resources maintain compliant attributes.

```sentinel
import "tfstate"

# Allowed values for resource attributes
allowed_values = {
 "aws_instance.example.instance_type": ["t2.micro", "t3.micro"],
 "aws_security_group.example.description": ["Allow HTTP", "Allow HTTPS"],
}
```

### Using `tfrun`
The `tfrun` import examines metadata such as cost estimates or workspace attributes.

```sentinel
import "tfrun"

# Calculate proposed monthly cost
delta_monthly_cost = decimal.new(tfrun.cost_estimate.delta_monthly_cost)
```

---

## Key Takeaways

Sentinel enables automation and enforcement of policies within Terraform workflows, improving security, compliance, and standardization. Testing with mocks ensures reliability, while functions and imports provide flexibility to tailor policies to specific needs. Starting with **Advisory** enforcement allows for a smooth transition to stricter levels as confidence grows. By integrating deeply with Terraform, Sentinel empowers organizations to adopt Policy-as-Code practices effectively.