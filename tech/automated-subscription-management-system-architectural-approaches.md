# Automated Subscription Management System: Architectural Approaches
Subscription, also referred to as paid subscription or subscription-based business model, is a framework that fosters a long-term relationship between businesses and users. Users pay periodically, while businesses continually provide products or services with specific features and value. This model has found extensive applications in industries such as Software-as-a-Service (SaaS), streaming media, subscription box services, and manufacturing. Key aspects of subscription-based models include recurring consumption, flexible pricing, service upgrades or downgrades, pausing and resuming services, and usage-based billing. This essay explores the application of various architectural styles in building an automated subscription management system.

## Overview of the System
The system is built on cloud computing infrastructure and operates on AWS, with Java as the primary development language. It is a critical commercial project where the choice of architectural style significantly impacts its success. Furthermore, the system is expected to handle future changes effectively, emphasizing scalability and maintainability. Therefore, selecting the appropriate architectural styles is of paramount importance.

## Architectural Styles
There are five commonly adopted architectural styles: data flow, call-and-return, virtual machine, repository, and independent components. Each has distinct characteristics and application domains.
- **Data Flow Style**: Processes data sequentially along a predefined flow.
- **Repository Style**: Centralized around data, where all operations revolve around a data repository.
- **Call-and-Return Style**: Features explicit invocation relationships between components.
- **Independent Components Style**: Components are loosely coupled, communicating asynchronously or via event triggers. Examples include process communication and event-driven systems, which facilitate software reuse and evolution but sacrifice centralized computational control.
- **Virtual Machine Style**: Provides a custom set of rules for users to develop components, often ensuring cross-platform compatibility.

Upon analyzing the definitions, use cases, advantages, and disadvantages of these styles, it becomes clear that each addresses specific problem types. For complex systems, employing a combination of styles is often the most effective strategy. In this project, the automated subscription management system incorporates hierarchical structure from the call-and-return style, process communication from the independent components style, and interpreters from the virtual machine style. Below is a detailed analysis of their application.

## Call-and-Return Style: Hierarchical Structure
The hierarchical structure organizes components into layers, where each layer serves the one above it and uses the services of the one below it. Modifying one layer impacts, at most, the two adjacent layers, promoting software reuse. The system architecture is divided into three layers:
- **Presentation Layer**: Handles user interactions, input validation, and data visualization. Technologies used include React, JavaScript, and HTML5.
- **Business Logic Layer**: Implements core functionalities using Java and Spring Boot, including RESTful APIs, payment gateway integration, and key modules like user management, pricing management, subscription management, automated invoicing, payment handling, and revenue recognition. These modules comprehensively support subscription lifecycle management, including designing subscription plans, activating users, managing pricing, automating billing, processing changes, and tracking revenue.
- **Data Layer**: Manages data access and storage using the Hibernate ORM framework and AWS Aurora MySQL database cluster.

## Independent Components Style: Process Communication
In this style, components operate as independent processes communicating via message passing, which can be synchronous or asynchronous. In this project, process communication is employed for non-real-time business functions like the "Receivables vs. Received Reports."

When a subscription is created, a "receivable" is generated, representing the amount owed based on the subscription duration, product pricing, and pricing model. Upon invoicing and payment, a "received" entry is recorded. The "Receivables vs. Received Report" compares these to ensure all receivables are collected. Typically generated near the end of an accounting period, these reports do not require real-time updates.

To handle this, a message queue (ActiveMQ) is utilized. When a subscription is created or modified, the system sends a message to the queue with relevant details, then promptly returns a response to the user. A consumer processes the message asynchronously, updates the database, and reflects the changes in the report. This approach reduces response time and supports horizontal scaling by adding more consumers to handle increased message volumes.

## Virtual Machine Style: Interpreter
Interpreters allow dynamic modification of rules through script updates or configuration changes, offering exceptional flexibility and maintainability. In this project, interpreters are used to develop a custom report generation system, enabling users to define templates via dropdowns and drag-and-drop interfaces.

### Technical Implementation
1. Domain-Specific Language (DSL): A DSL is designed for defining report templates, including titles, data fields, conditions, and aggregation methods.
2. Interpreter Components:
    - Template Interpreter: Parses report titles and content.
    - Field Interpreter: Extracts data fields from the database.
    - Condition Interpreter: Applies user-defined filters to display relevant data.
    - Aggregation Interpreter: Executes aggregation functions on data as specified by the user.

Users can easily customize their reports, while the system interprets the DSL and generates the desired output dynamically, achieving high levels of customizability and flexibility.

## Conclusion
By integrating multiple architectural styles, the automated subscription management system effectively addresses diverse business needs. The hierarchical structure supports modularity and maintainability, process communication enhances scalability and responsiveness for non-real-time tasks, and interpreters enable dynamic customization. This hybrid approach ensures the system's robustness, scalability, and adaptability, making it well-suited for complex and evolving subscription-based businesses.

