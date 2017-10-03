## CI/CD for microservices

One of the main motivations for building microservices is to enable a faster release cycle. 

Update services independently


If the team building service "A" wants to release an update, they should not be held up because changes to service "B" are waiting to be merged, tested, and deployed. There should not be a long "release train" where every team has to get in line.

The release pipeline must be automated and highly reliable, so that the risks of deploying updates are minimized. If you are releasing to production daily or multiple times a day, regressions or service disruptions must be very rare.

At the same time, if a bad update does get deployed, you must have a reliable way to quickly roll back to a previous version of a service.

Enable a decentralized DevOps process. Every team should have the ability to deploy an update to production. That doesn't mean that every team member has permissions to do so. But having a centralized "release manager" role can reduce the velocity of deployments. The more the CI/CD process is automated and reliable, the less there should be a need for a central authority. (That said, you might have different policies for releasing major feature updates versus minor bug fixes.)

Challenges

- No shared code base. Each team is responsible for building its own service. 
- Some teams may even use separate code repositories.
- Multiple languages and frameworks. The build process must be flexible enough that every team can adapt it for their choice of language or framework.
    - Create a container for the build environment. The build server runs the container and the container builds the artifacts. The build server does not have to be configured with multiple build tools.
- Performing integration testing of a service in a production-like environment with other live services.
- Running a full production cluster can be expensive, so it's unlikely that every team will be able to run its own full cluster just for testing. 

Our approach:

For local development and testing, use Docker to run the container. As part of this process, you may need to run other containers that have mock services or test databases, that you need for local testing. Use Docker Compose to coordinate this. Another option is to use minikube. You are also running unit tests during this phase.

When the code is ready, open a PR and merge into master. This will start a job on the build server that

- Builds the code assets. 
- Runs unit tests
- Builds an image
- Pushes the image to a container registry
- Updates the test cluster with the new image to run integration tests.



When the image is ready to go into production,

- Update the deployment files (Kubernetes YML files, Heml charts) to specify the latest image
- Apply the update to the production cluster.
 
You may want to create a separate container registry for production, that will only hold known-good images. If you combine this approach with semantic versioning, then you can always find the last-known-good image in case you need to roll back. 


 
 







