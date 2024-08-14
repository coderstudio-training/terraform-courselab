# K3s and Terraform Exercises

These exercises are designed to reinforce your understanding of the local K3s and Terraform tutorial. Complete these tasks to practice and expand upon the concepts covered in the main tutorial.

## Exercise 1: Modify the Frontend

Modify the React frontend to include a "Clear All Messages" button. This button should make an API call to a new endpoint on the backend that clears all messages from the database.

1. Update the React component in `frontend/src/App.js`.
2. Create a new API endpoint in the backend `server.js` file.
3. Update the Terraform configuration to reflect any changes (if necessary).

## Exercise 2: Add Environment Variables

Instead of hardcoding the database credentials, use environment variables in the backend application.

1. Modify the backend `server.js` to use environment variables for database connection.
2. Update the Terraform configuration to pass these environment variables to the backend deployment.

## Exercise 3: Implement Health Checks

Add liveness and readiness probes to your backend deployment.

1. Implement a `/health` endpoint in your backend application.
2. Update the Terraform configuration to include liveness and readiness probes for the backend deployment.

## Exercise 4: Scale the Backend

Modify the Terraform configuration to allow easy scaling of the backend deployment.

1. Update the Terraform configuration to use a variable for the number of backend replicas.
2. Test scaling up and down using Terraform.

## Exercise 5: Add Redis Caching

Integrate Redis as a caching layer between your backend and database.

1. Add a Redis deployment and service using Terraform.
2. Modify the backend to use Redis for caching frequently accessed data.
3. Update the Terraform configuration to include the necessary environment variables for Redis connection.

## Bonus Exercise: Implement Ingress

If you're feeling adventurous, implement an Ingress resource to route traffic to your frontend and backend services.

1. Install an Ingress controller in your K3s cluster.
2. Create an Ingress resource using Terraform.
3. Update your application to work with the new Ingress routing.

Remember to test your changes thoroughly and ensure that your Terraform configuration remains valid throughout these exercises.

---

Feeling stuck? Click [here](/exercises/solution-part-1/){:target="_blank"} to see the solution
