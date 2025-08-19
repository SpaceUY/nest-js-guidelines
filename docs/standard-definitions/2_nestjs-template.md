# NestJS Template

We provide a [NestJS Template](https://github.com/SpaceUY/NestJS-Template) as a starting point for backend projects. The main goal of this template is to offer a development guide and pre-built implementations of common features used across multiple projects. By leveraging the [planetary tool](https://github.com/SpaceUY/planetary), these implementations can be easily exported and integrated into new backend projects.

**Adapter Pattern**

The template makes use of the Adapter Pattern to provide flexibility and extensibility for different integrations. This is achieved by defining a common abstract class for each module (such as Email, Cloud Storage, or Push Notifications), and then implementing provider-specific subclasses that extend this abstract class. Each subclass handles the integration with a particular provider (e.g., SendGrid, Resend, AWS SES for Email), allowing you to switch or add providers with minimal changes to the rest of your codebase.

The adapter pattern ensures a consistent interface for each module, making it easy to use and extend integrations as needed across different projects.

**Available adaptable modules:**
- **Email:** Includes integration with SendGrid. [email module](https://github.com/SpaceUY/NestJS-Template/pull/14)
- **Cloud Storage:** Includes integration with S3 ([cloud-storage module](https://github.com/SpaceUY/NestJS-Template/tree/master/src/cloud-storage)).
- **Push Notifications:** Includes integration with Expo ([push-notification module](https://github.com/SpaceUY/NestJS-Template/tree/master/src/push-notification)).

The template is currently being updated to align with the latest company standards. We encourage contributions! To participate, join the NestJS chapter calls to suggest new implementations or to take ownership of a feature you would like to add.

> **Note:** Before using the template to start a new project, please ensure it is up to date with the latest versions, standards and features.

**Coming soon:**
- Auth0 integration
- TypeORM implementation
- Resend and AWS SES integration for emails
