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
#### DNS round-robin, Gateway dual VIP:    
#### Network card upgrade, 10G -> 25G：      
#### Split internal and external traffic to different network cards：     

#### Layer 4 load balancing  
[ref](https://www.kawabangga.com/posts/5301)   
#### Service optimization, optimize existing network requests.   



## Ways to Experiment: Using Stress-Testing Tools   
Comparison of Using Different Stress-Testing Tools:   

Tools to Consider:     
- python custom script       
- go-stress-testing    
- apache bench    
- jmeter    

