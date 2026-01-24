# Web Applications Firewall 

## F5 Advanced WAF

**F5 WAF (Web Application Firewall)**, currently known as **F5 Advanced WAF (AWAF)** and formerly as **BIG-IP Application Security Manager (ASM)**, is a security solution designed to protect web applications and APIs from various application-layer (Layer 7) threats.  

It acts as a reverse proxy that filters and monitors HTTP/HTTPS traffic between the internet and a web application. Key functions include:   
- **Vulnerability Protection**: Blocks common attacks like SQL injection, cross-site scripting (XSS), and other OWASP Top 10 risks.
- **Advanced Defense**: Uses machine learning and behavioral analytics to identify and mitigate automated bot attacks, credential stuffing, and application-layer DoS (Denial-of-Service).
- **Data Safeguarding**: Features like "Data Guard" mask sensitive information (e.g., credit card numbers) in server responses to prevent data leakage.
- **Deployment Versatility**: Available as hardware appliances, virtual editions for cloud environments (AWS, Azure, Google Cloud), or as a SaaS-delivered service via F5 Distributed Cloud.  

#### Programming and Scripting Languages

- **Tcl (Tool Command Language)**: The core scripting language for F5 devices is **Tcl**, used to write **iRules**. iRules allow administrators to programmatically inspect, intercept, and modify traffic passing through the WAF in real-time.
- **Python**: Primarily used for external automation and management. F5 provides SDKs and modules that allow developers to interact with the WAF’s REST API using Python for tasks like automated policy deployment and configuration in CI/CD pipelines.
- **JavaScript (Node.js)**: Newer F5 platforms support iRules LX, which allows developers to use Node.js and JavaScript to extend the functionality of the WAF beyond what standard Tcl iRules can do.
- **YAML/JSON**: Used for declarative configuration, where the WAF's security policies and settings are defined in structured data files for automated, consistent deployments.  

## AWS WAF

**AWS WAF (Web Application Firewall)** is a cloud-native, fully managed security service that protects web applications and APIs from common internet threats and malicious bots. It is not a software you install; rather, it is a configuration-based service that integrates directly with AWS resources like **Application Load Balancers (ALB)**, **Amazon CloudFront**, and **Amazon API Gateway**. 


AWS WAF uses Web **Access Control Lists (Web ACLs)** to define security rules. These rules inspect incoming HTTP/HTTPS traffic and can take actions such as `Allow`, `Block`, `Count` (for monitoring), or `Challenge` (using CAPTCHA) based on specific criteria like IP addresses, HTTP headers, or request body content.  

Unlike the F5 WAF, which uses Tcl for scripting (iRules), AWS WAF is primarily language-agnostic for the end user and does not require writing custom code to operate:
- **Rule Language (JSON/YAML)**: You define WAF rules using a Declarative JSON or YAML syntax. While you can use a visual builder in the AWS Console, complex rules are often managed as JSON files in automated environments.
- **Automation (Python, JavaScript, etc.)**: For managing and deploying rules programmatically, you can use the AWS SDKs or the AWS CLI. These support various languages, with Python and JavaScript being the most common for automation and serverless (Lambda) integration.
- **Internal Development**: While not exposed to users, the underlying service for many AWS data planes is increasingly being developed using Rust for performance and safety, following a legacy of Java and C++.
- **Edge Extensions (Lambda@Edge)**: If you need to write custom logic that goes beyond standard WAF rules, you can use JavaScript (Node.js) or Python via Lambda@Edge to intercept and modify requests at the global edge.  

### Key Features for 2026  

- **Managed Rule Groups**: Pre-configured rules maintained by AWS or marketplace sellers to protect against the OWASP Top 10.
- **Bot Control**: Advanced behavioral analysis to identify and mitigate automated bot traffic, such as `scrapers` or `crawlers`.
- **Fraud Control**: Specialized rule groups to prevent **Account Takeover (ATO)** and **Account Creation Fraud**.
- **Real-Time Monitoring**: Native integration with **Amazon CloudWatch** for traffic metrics and logging. 

### Integration

#### Integration with Application Load Balancer (ALB)

- **Direct Association**: You can link a Web ACL to an ALB via the AWS WAF console, the ALB console, or programmatically.
- **One-Click Integration**: The ALB console offers a "one-click" setup that automatically creates and associates a Web ACL with AWS-recommended default protections.
- **Fail Open/Close**: By default, if the ALB cannot reach AWS WAF, it returns an `HTTP 500 error`. You can change this to "`fail open`" to allow traffic to pass even if the WAF is unavailable.  

#### Integration with Ingress Controller (Kubernetes/EKS)

- **AWS Load Balancer Controller**: In an EKS environment, the AWS Load Balancer Controller manages the creation of an ALB for your Kubernetes Ingress.
- **Declarative Configuration**: You associate WAF with your Ingress by adding specific annotations to your Kubernetes Ingress manifest (e.g., `alb.ingress.kubernetes.io/wafv2-acl-arn: <ARN>`).
- **Auto-reconciliation**: The controller automatically detects these annotations and updates the ALB's association with the specified Web ACL.  

#### Integration with Amazon API Gateway

- **REST APIs**: You associate a Web ACL with a specific API Stage.
- **Regional Enforcement**: **WAFv2 Web ACL**s must be "Regional" to work with API Gateway (as opposed to "Global" used for CloudFront).
- **Supported Types**: Native integration is primarily for REST APIs; as of 2026, many HTTP API features are also supported in specific regions.
- **Configuration**: Association is typically done through the API Gateway console under the "`Stages`" settings or via Infrastructure as Code (Terraform/CloudFormation) using the `aws_wafv2_web_acl_association` resource.  

## Amazon CloudFront

Amazon CloudFront is a globally distributed **Content Delivery Network (CDN)** service that accelerates the delivery of static and dynamic web content (like HTML, images, and video) by serving it from a network of over 600 **Points of Presence (PoPs)** across 100+ cities in 50+ countries

### Core Functionality

- **Edge Caching**: When a user requests content, CloudFront routes them to the edge location with the lowest latency. If the content is cached there (a "`cache hit`"), it is delivered immediately. If not (a "`cache miss`"), CloudFront retrieves it from an origin (such as Amazon S3, EC2, or a custom server) and caches it for future requests.
- **Regional Edge Caches**: Larger caching nodes that sit between edge locations and your origin. They store content that isn't popular enough for every edge location, reducing the number of requests that must travel all the way back to your origin server.  

#### Key Features for 2026
- **Programmable Edge (Edge Computing)**: CloudFront Functions: Lightweight JavaScript functions for high-scale, simple tasks like header manipulation or URL redirects, executing in under 1 millisecond.
- **Lambda@Edge**: Powerful serverless compute for complex logic like server-side rendering or custom authentication, supporting Node.js and Python.
- **Advanced Security**:
- **Viewer mTLS**: Launched in late 2025, this allows for mutual TLS authentication, ensuring both the client and the server verify each other's identity before exchanging sensitive data.
- **Native Integrations**: Direct protection from **AWS Shield (DDoS)** and **AWS WAF (Layer 7 application security**) at the edge.
- **Continuous Deployment**: Allows you to test two identical environments (blue/green) and roll out releases gradually to a small percentage of users without DNS changes.  
