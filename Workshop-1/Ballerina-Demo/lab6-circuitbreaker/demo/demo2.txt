// To compensate for slowness use circuit breaker
// To run it:
// ballerina run demo_circuitbreaker.bal --config twitter.toml
// To invoke:
// curl -X POST localhost:9090
// Invoke many times to show how circuit breaker works

import ballerina/http;
import wso2/twitter;
import ballerina/config;

// Change the endpoint initialization to add timeout (half-send)
// and circuit breaker logic
endpoint http:Client homer {
 url: "http://www.simpsonquotes.xyz",
 circuitBreaker: {
     failureThreshold: 0,
     resetTimeMillis: 3000,
     statusCodes: [500, 501, 502]
 },
 timeoutMillis: 500
};

endpoint twitter:Client tw {
 clientId: config:getAsString("clientId"),
 clientSecret: config:getAsString("clientSecret"),
 accessToken: config:getAsString("accessToken"),
 accessTokenSecret: config:getAsString("accessTokenSecret"),
 clientConfig: {} 
};

@http:ServiceConfig {
 basePath: "/"
}
service<http:Service> hello bind {port: 9090} {

 @http:ResourceConfig {
     path: "/",
     methods: ["POST"]
 }
 hi (endpoint caller, http:Request request) {
     http:Response res;
     
     // use var as a shorthand for http:Response | error union type
     // Compiler is smart enough to use the actual type
     var v = homer->get("/quote");

     // match is the way to provide different handling of error vs normal output
     match v {
         http:Response hResp => {

             // if proper http response use our old code
             string payload = check hResp.getTextPayload();
             if (!payload.contains("#ballerina")){payload=payload+" #ballerina";}
             twitter:Status st = check tw->tweet(payload);
             json myJson = {
                 text: payload,
                 id: st.id,
                 agent: "ballerina"
             };
             res.setPayload(untaint myJson);
         }
         error err => {
             // this block gets invoked if there is error or if circuit breaker is Open
             res.setPayload("Circuit is open. Invoking default behavior.\n");
         }
     }
     _ = caller->respond(res);
 }
}