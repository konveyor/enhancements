# Konveyor Project Roadmap
 * Konveyor is a CNCF project focused on accelerating application modernization to cloud native technologies in a safe and predictable manner at scale
 	* Related documents for more background on Konveyor:
 		* [Konveyor Project Charter](https://github.com/konveyor/community/blob/main/Charter.md)
		* Vision for Konveyor's [Unified Experience](https://github.com/konveyor/enhancements/tree/master/enhancements/unified_experience)
		* [Konveyor Application Modernization and Migration Guide](https://github.com/konveyor/methodology)
	* Where to file new RFEs? -> https://github.com/konveyor/enhancements/issues/new
	* Read our [enhancements](https://github.com/konveyor/enhancements) to learn more about design choices and how features are being implemented

## Near-term: 2023 Q3-Q4

* Theme: Establish [Steps 1->3 of Unified Experience: 'Surface Information -> Make Decisions -> Plan Work'](https://github.com/konveyor/enhancements/tree/master/enhancements/unified_experience#step-1-surface-information)
	* **Multi-language analysis support** 
		* A new analysis engine, [analyzer-lsp](https://github.com/konveyor/analyzer-lsp), is being developed to leverage the [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) to make additional languages easier to integrate.
			* enhancement: [Konveyor Analysis: Multi-language analyzer leveraging Language Server Protocol (LSP)](https://github.com/konveyor/enhancements/tree/master/enhancements/tackle/tackle-lsp-query-engine)
		* Intent is to replace [Windup](https://github.com/windup/windup) for Java source code analysis with [analyzer-lsp](https://github.com/konveyor/analyzer-lsp) 
    	* Add ability for Golang source code analysis via analyzer-lsp

	* **Allow dynamic reporting of analysis data** and aggregation across the application portfolio (enhancement: [Dynamic reports enhancement](https://github.com/konveyor/enhancements/pull/111))
  		* Establish ability for machine readable output from analyzer-lsp to aid the Hub to aggregate analysis information across the application portfolio (enhancement: [Analyzers Report Format](https://github.com/konveyor/enhancements/tree/master/enhancements/tackle/analyzers-report-format)
	
	* **New Assessment module** to allow greater customization of questionnaires and usage (enhancement: [New assessment module enhancement](https://github.com/konveyor/enhancements/pull/136))
		* Introduction of '[Archetypes](https://github.com/konveyor/enhancements/pull/136/files#diff-2d249d6ed5064c180ca1a72da0658dc4322786d504739df84e55754ce83d6c86R199)'
		* Intent is to replace Pathfinder with this new module to allow more flexibility

* Theme: Project health
	* Establish e2e test strategy 
	* Establish test framework for [rulesets](https://github.com/konveyor/rulesets)
	* Continue creating a library of [examples-applications](https://github.com/konveyor/example-applications/tree/main)
	* Establish [Migration Experience User Group](https://github.com/konveyor/community/tree/main/ug-migration-experience) to facilitate sharing of use-cases and rules throughout community.

## Mid term: 2024 Q1-Q2 
* Theme: Expand multi-language static code analysis capabilities
    * Establish C# analysis for dotnet ([analyzer-dotnet-provider](https://github.com/konveyor/analyzer-dotnet-provider)), emphasis on usecase of .Net Framework -> .Net migrations
	    * enhancement: [dotnet-provider](https://github.com/konveyor/enhancements/pull/128)

* Theme: Begin to explore improvements to [Step 4 of Unified Experience: 'Do Work'](https://github.com/konveyor/enhancements/tree/master/enhancements/unified_experience#step-4-do-work)
	* Evaluate if we need to orchestrate execution of multiple components in a Pipeline like approach
		* Possible exploration of Tekton for improving orchestration of components

    * Generate migration assets to aid a Migrator modernizing an application
	    * Considering examples such as
		    * Guide containerization of a traditional application
		    * Guide containerization of a Wildfly application to leverage application analysis information to recommend Galleon layers to enable in the image
		    * Generate Kubernetes resource manifests so the application may be deployed
		    * Generate resources to enable a GitOps workflow such as ArgoCD
		    * etc.. more to explore

* Theme: Continue on expanding [Rulesets](https://github.com/konveyor/rulesets)
    * Considering examples related to:
        * .Net Framework -> .Net
        * Wildfly
        * Tomcat
        * Quarkus
        * Netflix OSS -> Kubernetes
        * Service Mesh adoption
        * Serverless
        * and other areas identified from the [Migration Experience User Group](https://github.com/konveyor/community/tree/main/ug-migration-experience)
* Theme: Project Health
	* Improve Konveyor user documentation and revisit integrating https://konveyor.github.io/ into https://konveyor.io

## Long term: 2024 Q2+

* Theme:  Expand multi-language capabilities
    * Add in capability to analyze Javascript, Typescript, Python, Ruby, Rust, etc
    * Continue to extend capabilities in all providers (richer queries, detect more complicated use-cases)
    
* Theme:  Experience improvements
    * Improve the experience of application import
        * [Repository crawler for application import](https://github.com/konveyor/enhancements/issues/122)
        * [Detect language & frameworks being used in repositories on import](https://github.com/konveyor/enhancements/issues/131) 

    * Improve the experience of executing 'AddOns'
        * [Enable the UI to run arbitrary AddOns](https://github.com/konveyor/enhancements/issues/99)

* Theme: Continue to explore new use-cases in [Konveyor Ecosystem](https://github.com/konveyor-ecosystem)
	* Explore solutions to aid in monolith to microservice
	* Improve resource generation
	* Explore k8s workload introspection to inform possible issues between versions of Kubernetes
	* Expand refactoring scenarios, i.e. explore patterns of leveraging knowledge from static code analysis to aid in automated refactoring to address identified issues (i.e. analyzer-lsp report -> openrewrite recipes to 'fix' code)
	* Research if/how AI may be applied to modernization (experiments under https://github.com/konveyor-ecosystem/MLAssist)

* Theme: Integration with other services
    * [Janus (backstage.io) Integration #134](https://github.com/konveyor/enhancements/issues/134)

## Summary of Konveyor Releases
See [konveyor/operator/releases](https://github.com/konveyor/operator/releases) for a list of all Konveyor releases or in [operatorhub here](https://operatorhub.io/operator/konveyor-operator)
 * __0.1.0__: April 2023
	* The effort that was Tackle 2.0 is expanded to become the first release of Konveyor with the new vision based on the [Unified Experience](https://github.com/konveyor/enhancements/tree/master/enhancements/unified_experience).  
 * __0.2.0__: June 2023 (_alpha released May 2023_)
	* The ability to break up and plan units of work via Migration Waves is introduced
		* enhancement: [Migration Waves](https://github.com/konveyor/enhancements/tree/master/enhancements/migration-waves)
		* enhancement: [Tackle Hub integration with Jira](https://github.com/konveyor/enhancements/tree/master/enhancements/tackle/tackle-jira-integration)
 * __0.3.0__: Planned Q3 2023 (_alpha released August 2023_)
	* Multi-language analysis capability is introduced via [analyzer-lsp](https://github.com/konveyor/analyzer-lsp) that leverages [Language Server Protocol](https://microsoft.github.io/language-server-protocol/).  
		* enhancement: [Konveyor Analysis: Multi-language analyzer leveraging Language Server Protocol (LSP)](https://github.com/konveyor/enhancements/tree/master/enhancements/tackle/tackle-lsp-query-engine)
			* [Windup](https://github.com/windup/windup) is replaced by [analyzer-lsp](https://github.com/konveyor/analyzer-lsp) 
			* Languages supported are Java and Golang. 
	* Dynamic reporting and aggregation of analysis data across the entire application portfolio is now possible
		* enhancement: [Dynamic reports enhancement](https://github.com/konveyor/enhancements/pull/111)
		* enhancement: [Analyzers Report Format](https://github.com/konveyor/enhancements/tree/master/enhancements/tackle/analyzers-report-format)
	* [Active development, not in 0.3.0-alpha]: New Assessment Module to replace Pathfinder
		* enhancement: [New assessment module enhancement](https://github.com/konveyor/enhancements/pull/136)
 ---

For more information and to get involved, please join the [konveyor-community@googlegroups.com](https://groups.google.com/g/konveyor-community) mailing list and/or attend a [Konveyor Community Meeting](https://github.com/konveyor/community/tree/main#meetings).

This roadmap is a living document and will evolve as the project progresses. Stay tuned for updates and announcements as we work together to revolutionize application modernization to Kubernetes!

