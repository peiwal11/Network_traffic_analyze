# Network Traffic Analyze OUTLINE  
 

## Current Network Configuration:  
(diagram , 3 instance, kong and minio with same ip....)

Objective:  determine the system's upper bounds and the proportion of kong's outbound traffic that is dedicated to client response  
 
#### Calculations:  
- assumptions
- the proportion of kong's outbound traffic that is dedicated to client response



#### Experiment Observations:
- experiment configuration (1Gbps , filesize:, how many req, from how many concurrent connections   
- observations   
- discrepency between assumptions and experiment results   

### Under such conditions, Possible Ways to Handle Increased Traffic:   
#### Two VIPs (Virtual IPs):   
   
#### Separate Network Interfaces for Minio and Kong:     

#### Load Balancing on Different Layers:   
[ref](https://www.kawabangga.com/posts/5301)
## Ways to Experiment: Using Stress-Testing Tools   
Comparison of Using Different Stress-Testing Tools:   

Tools to Consider:     
- python custom script       
- go-stress-testing    
- apache bench    
- jmeter    

