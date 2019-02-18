## Modifying the Pong Microservice

The ping service is hardcoded to use bob and bobpwd as the credentials to authenticate against the name service. You’ll make these credentials configurable. Replace the PingResource class in the ping/src/main/java/io/openliberty/guides/ping/PingResource.java file with the following:

```java
package io.openliberty.guides.ping;

import java.net.MalformedURLException;
import java.net.URL;
import java.net.UnknownHostException;
import java.util.Base64;

import javax.enterprise.context.RequestScoped;
import javax.inject.Inject;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.PathParam;
import javax.ws.rs.ProcessingException;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

import org.apache.commons.lang3.exception.ExceptionUtils;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.eclipse.microprofile.rest.client.RestClientBuilder;

import io.openliberty.guides.ping.client.NameClient;
import io.openliberty.guides.ping.client.UnknownUrlException;

@RequestScoped
@Path("")
public class PingResource {

    @Inject
    @ConfigProperty(name = "USERNAME")
    private String username;

    @Inject
    @ConfigProperty(name = "PASSWORD")
    private String password;

    @GET
    @Path("/{hostname}")
    @Produces(MediaType.TEXT_PLAIN)
    public String getContainerName(@PathParam("hostname") String host) {
        try {
            NameClient nameClient = RestClientBuilder.newBuilder()
                            .baseUrl(new URL("http://" + host + ":9080/api"))
                            .register(UnknownUrlException.class)
                            .build(NameClient.class);
            nameClient.getContainerName(getAuthHeader());
            return "pong\n";
        } catch (ProcessingException ex) {
            // Checking if UnknownHostException is nested inside and rethrowing if not.
            if (this.isUnknownHostException(ex)) {
                System.err.println("The specified host is unknown");
                ex.printStackTrace();
            } else {
                throw ex;
            }
        } catch (UnknownUrlException ex) {
            System.err.println("The given URL is unreachable");
            ex.printStackTrace();
        } catch (MalformedURLException ex) {
            System.err.println("The given URL is not formatted correctly.");
            ex.printStackTrace();
        }
        return "Bad response from " + host + "\nCheck the console log for more info.";
    }

    private boolean isUnknownHostException(ProcessingException ex) {
        Throwable rootEx = ExceptionUtils.getRootCause(ex);
        return rootEx != null && rootEx instanceof UnknownHostException;
    }

    private String getAuthHeader() {
        String usernamePassword = username + ":" + password;
        String encoded = Base64.getEncoder().encodeToString(usernamePassword.getBytes());
        return "Basic " + encoded;
    }
}
```

The changes introduced here use MicroProfile Config and CDI to inject the value of the environment variables USERNAME and PASSWORD into the PingResource class. Then the hardcoded values are removed from those same properties.
