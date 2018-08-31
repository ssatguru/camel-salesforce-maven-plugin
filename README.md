# Customized version of the camel-salesforce-maven-plugin

This is a customized version of the camel-salesforce-maven-plugin (version 2.18.1) [https://github.com/apache/camel/tree/master/components/camel-salesforce/camel-salesforce-maven-plugin](https://github.com/apache/camel/tree/master/components/camel-salesforce/camel-salesforce-maven-plugin)  

It fixes the an issue with the original plugin described in this ticket [add support for lookup field using an sObject external id](https://issues.apache.org/jira/browse/CAMEL-10193) and further explained below.  

A SalesForce object can have a field of type called "lookup".  
A record in such a SalesForce object can use that field to reference a record in another SalesForce object.  
The value of this field can be a string in which case it should be the record id of the other record or it could be an object in which case it should be a json object containing the other Object's external id field name and its value in that other record.  
 
Doing lookup/reference by external id is very useful specially for insert/upsert operations as otherwise one has to maintain record id of each  record in the other Object.    
 
See here for more information on this   
https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/dome_upsert.htm  
Section "Upserting Records and Associating with an External ID"  

The original Camel SalesForce component supports lookup by record id but doesnot support lookup by the external id field name and value type.  

This customized version provides this additional capability.  

It does this by adding an additional element called "sObjectExternalIdMap" under the salesforce maven plugin "configuration" element.  

This provides mapping of a sObject to its external id field.  
Here is how it might look
```
<configuration>
	<sObjectExternalIdMap>
		<Account>BranchCode__c</Account>
		<Case>ExtField__c</Case>
		...

	</sObjectExternalIdMap>
</configuration>
```
So here Account's external id is `BranchCode__c`, Cases's external id is `ExtField__c` and so on.  
During processing the plugin checks if a field is a lookup field and if yes then what sObject does it look up and has the user provided a mapping for the sObject being looked up.  
If yes then it generates a Lookup class for that field and sets the type of the field to that class.  
If not then it sets the type to String (the default behavior as before).  

So, for example, if Contact does a lookup on Account via field AccountName then in the Contact DTO it sets AccountName field type to AccountLookup and also creates a class called AccountLookup.
```
public class AccountLookup implements Lookup{

    private String BranchCode__c;

    public void setBranchCode__c(String e){
    	this.BranchCode__c = e;
    }
    
    public String getBranchCode__c(){
    	return this.BranchCode__c;
    }
    ...

}
```
Further if the lookup field is a custom field and thus ends with "_c" , the plugin replaces "c" with "_r"  

So if Contact did a lookup on Account using a custom field called say "CustomField_c" then in the Contact DTO it changes the name of the field to "CustomField_r" and of course sets the type to AccountLookup.  

On serialization these generate  
```
{
	...
	
	AccountName : {
		BranchCode__c:"123"	
	},

	CustomField__r : {
		BranchCode__c:"456"	
	}
	...
}
```

This is also backward compatible, so if no "sObjectExternalIdMap" is provided under "configuration" element the plugin works exactly as before.  
  
Below is the standard documentaion of the original plugin.  
  
# Maven plugin for camel-salesforce component #

This plugin generates DTOs for the [Camel Salesforce Component](https://github.com/dhirajsb/camel-salesforce). 

## Usage ##

The plugin configuration has the following properties.

* clientId - Salesforce client Id for Remote API access
* clientSecret - Salesforce client secret for Remote API access
* userName - Salesforce account user name
* password - Salesforce account password (including secret token)
* version - Salesforce Rest API version, defaults to 25.0
* outputDirectory - Directory where to place generated DTOs, defaults to ${project.build.directory}/generated-sources/camel-salesforce
* includes - List of SObject types to include
* excludes - List of SObject types to exclude
* includePattern - Java RegEx for SObject types to include
* excludePattern - Java RegEx for SObject types to exclude
* packageName - Java package name for generated DTOs, defaults to org.apache.camel.salesforce.dto.

Fro obvious security reasons it is recommended that the clientId, clientSecret, userName and password fields be not set in the pom.xml. 
The plugin should be configured for the rest of the properties, and can be executed using the following command:

	mvn camel-salesforce:generate -DcamelSalesforce.clientId=<clientid> -DcamelSalesforce.clientSecret=<clientsecret> -DcamelSalesforce.userName=<username> -DcamelSalesforce.password=<password>

The generated DTOs use Jackson and XStream annotations. All Salesforce field types are supported. Date and time fields are mapped to Joda DateTime, and picklist fields are mapped to generated Java Enumerations. 
