# UI5Con2019
## Demo steps

namespace: be.wl
name: PersonSkills
service: ZGW_UI5CON_SRV

Remove /FG2 from service url in manifest.json

 - Add CoreService
 
        sap.ui.define([
        	"sap/ui/base/Object"
        ], function (Object) {
        	"use strict";
        
        	return Object.extend("be.wl.PersonSkills.service.CoreService", {
        		constructor: function (model) {
        			Object.call(this);
        			if (model) {
        				this.setModel(model);
        			}
        		},
        		setModel: function (model) {
        			this.model = model;
        		},
        		odata: function (url) {
        			var me = this;
        			var core = {
        				ajax: function (type, url, data, parameters) {
        					var promise = new Promise(function (resolve, reject) {
        						var args = [];
        						var params = {};
        						args.push(url);
        						if (data) {
        							args.push(data);
        						}
        						if (parameters) {
        							params = parameters;
        						}
        						params.success = function (result, response) {
        							resolve({
        								data: result,
        								response: response
        							});
        						};
        						params.error = function (error) {
        							reject(error);
        						};
        						args.push(params);
        						me.model[type].apply(me.model, args);
        					});
        					return promise;
        				}
        			};
        
        			return {
        				'get': function (params) {
        					return core.ajax('read', url, false, params);
        				},
        				'post': function (data, params) {
        					return core.ajax('create', url, data, params);
        				},
        				'put': function (data, params) {
        					return core.ajax('update', url, data, params);
        				},
        				'delete': function (params) {
        					return core.ajax('remove', url, false, params);
        				}
        			};
        		},
        		http: function (url) {
        			var core = {
        				ajax: function (method, url, headers, args, mimetype) {
        					var promise = new Promise(function (resolve, reject) {
        						var client = new XMLHttpRequest();
        						var uri = url;
        						if (args && method === 'GET') {
        							uri += '?';
        							var argcount = 0;
        							for (var key in args) {
        								if (args.hasOwnProperty(key)) {
        									if (argcount++) {
        										uri += '&';
        									}
        									uri += encodeURIComponent(key) + '=' + encodeURIComponent(args[key]);
        								}
        							}
        						}
        						if (args && (method === 'POST' || method === 'PUT')) {
        							var data = {};
        							for (var keyp in args) {
        								if (args.hasOwnProperty(keyp)) {
        									data[keyp] = args[keyp];
        								}
        							}
        						}
        						client.open(method, uri);
        						
        						if (method === 'POST' || method === 'PUT') {
        							client.setRequestHeader("accept", "application/json");
        							client.setRequestHeader("content-type", "application/json");
        						}
        						for (var keyh in headers) {
        							if (headers.hasOwnProperty(keyh)) {
        								client.setRequestHeader(keyh, headers[keyh]);
        							}
        						}
        						if (data) {
        							client.send(JSON.stringify(data));
        						} else {
        							client.send();
        						}
        						client.onload = function () {
        							if (this.status == 200) {
        								resolve(this.response);
        							} else {
        								reject(this.statusText);
        							}
        						};
        						client.onerror = function () {
        							reject(this.statusText);
        						};
        					});
        					return promise;
        				}
        			};
        
        			return {
        				'get': function (headers, args) {
        					return core.ajax('GET', url, headers, args);
        				},
        				'post': function (headers, args) {
        					return core.ajax('POST', url, headers, args);
        				},
        				'put': function (headers, args) {
        					return core.ajax('PUT', url, headers, args);
        				},
        				'delete': function (headers, args) {
        					return core.ajax('DELETE', url, headers, args);
        				}
        			};
        		}
        	});
        }); 
 
 - Add PersonService

        sap.ui.define([
    	"./CoreService",
    	"sap/ui/model/Sorter" ], function (CoreService, Sorter) {
    	"use strict";
    
    	var PersonService = CoreService.extend("be.wl.PersonSkills.service.PersonService", {
    		constructor: function (model) {
    			CoreService.call(this, model);
    		}
    	});
    	return PersonService;});

 - Add get function for person

    		getPerson: function (id) {
    			var sObjectPath = this.model.createKey("/Persons", {
    				Id: id
    			});
    			var mParameters = {
    				urlParameters: {
    					$expand: "PersonHasSkills"
    				}
    			};
    			return this.odata(sObjectPath).get(mParameters);
    		}

 - Register service in component
 
		PERSON:"Person",
		
		this._oPersonService = new PersonService(this.getModel());
		
		
		getService: function (sService) {
			return this["_o" + sService + "Service"];
		},	
 - show promise advantage in controller 
	 - remove functions: 
		 - _onMetadataLoaded, onListUpdateFinished, _bindView, _onBindingChange
		
	 - add line in object matched when metadata finished:

            this.getOwnerComponent().oListSelector.selectAListItem("/" + sObjectPath);
    
 - add service to the detail controller
 
	    this.PersonService= this.getOwnerComponent().getService(this.getOwnerComponent().PERSON);
		
	
 - add service request in objectmatched fn

				this.getModel().metadataLoaded().then(() => {
    				var sObjectPath = this.getModel().createKey("Persons", {
    					Id: sObjectId
    				});
    				this.getOwnerComponent().oListSelector.selectAListItem("/" + sObjectPath);
    
    				this.PersonService.getPerson(sObjectId).then((result)=>{
    					this.setModel(new JSONModel(result.data),"pers");
    				});
    
    			});

		
 - bind person details
 
		{pers>/Firstname}
        items="{pers>/PersonHasSkills/results}"
        pers>

- add state
	- add base object

            sap.ui.define([
            	"sap/ui/base/Object",
            	"sap/ui/model/json/JSONModel"
            ], function (Object, JSONModel) {
            	"use strict";
            	return Object.extend("be.wl.PersonSkills.model.BaseObject", {
            		constructor: function (data) {
            			this.copyValues(data);
            
            			if (this.isState) {
            				this.getModel().attachPropertyChange((oProperty) => {
            					// this["update" + oProperty.getParameter("path")] && this["update" + oProperty.getParameter("path").substr(1)]();
            					var fChangeFunction = this.getChangeFunction(oProperty.getParameter("path"));
            					this.callChangeFunction(fChangeFunction, oProperty);
            					if (oProperty.getParameter("context")) {
            						fChangeFunction = this.getChangeFunction(oProperty.getParameter("context").getPath() + "/" + oProperty.getParameter("path"));
            						this.callChangeFunction(fChangeFunction, oProperty.getParameter("context").getObject(), oProperty);
            						//call parent
            						var sPath = oProperty.getParameter("context").getPath();
            						var sParent = sPath.split("/")[sPath.split("/").length - 1];
            						if (!isNaN(parseInt(sParent))) { //in case of integer it's probably an array and we need to go one level up
            							sPath = sPath.split("/").slice(0, sPath.split("/").length - 1).join("/");
            						}
            						var sSourcePath = sPath.split("/").slice(0, sPath.split("/").length - 1).join("/");
            						var oSource = (sSourcePath && oProperty.getParameter("context").getModel().getProperty(sSourcePath));
            						fChangeFunction = this.getChangeFunction(sPath);
            						this.callChangeFunction(fChangeFunction, (oSource || oProperty.getParameter("context").getObject()), oProperty);
            
            					}
            				}, this);
            			}
            		},
            		getChangeFunction: function (sPath) {
            			sPath = sPath.substr(0, 1) === "/" ? sPath.substr(1) : sPath;
            			return sPath.split("/").reduce(function (prev,
            				curr,
            				idx, array) {
            				if (idx === array.length - 1) {
            					return prev[curr + "Changed"];
            				}
            				return curr && curr.length > 0 && prev ? prev[curr] : prev;
            			}, this);
            		},
            		callChangeFunction: function (fChangeFunction, scope, args) {
            			fChangeFunction && fChangeFunction.apply(scope, args);
            		},
            		copyValues: function (data) {
            			if (data) {
            				for (var field in data) {
            					switch (typeof (data[field])) {
            					case "object":
            						if (data[field] && data[field]["results"]) {
            							this[field] = data[field]["results"];
            						}
            						break;
            					default:
            						this[field] = data[field];
            					}
            				}
            			}
            		},
            		getModel: function () {
            			if (!this.model) {
            				this.model = new JSONModel(this, true);
            				//this.model.setData(this);
            			}
            			return this.model;
            		},
            		updateModel: function (bHardRefresh) {
            			if (this.model) {
            				this.model.refresh(bHardRefresh ? true : false);
            			}
            		}
            	});
            }); 
	
	- create person state

            sap.ui.define(["../model/BaseObject"], function (BaseObject) {
            	"use strict";
            	var PersonState = BaseObject.extend("be.wl.PersonSkills.state.PersonState", {
            		constructor: function (oService) {
							this.data = {
								Person : {},
								display : true
							};
							this.PersonService = oService;
							BaseObject.call(this, {
								isState: true
							});
            		}	});
            	return PersonState;
            });


	- add getPerson to state

				getPerson: function (id) {
					return this.PersonService.getPerson(id).then((result) => {
						this.data.Person = result.data;
						this.data.display = true;
						this.updateModel();
						return this.data.Person;
					});
				}
			
	- add to component
	    
                this._oPersonState = new PersonState(this._oPersonService);
			
    			,getState: function (sState) {
    				return this["_o" + sState + "State"];
    			}
			
	- update view controller
		
    			this.PersonState = this.getOwnerComponent().getState(this.getOwnerComponent().PERSON);
    			this.setModel(this.PersonState.getModel(), "pers");
	- update view controller - on matched fn	
	
				this.PersonState.getPerson(sObjectId);
		
	- update view bindings ( pers + /Person )
	
				text="{pers>/Person/Firstname} {pers>/Person/Lastname}"
	
- add objects
    - Skill

                sap.ui.define([
                	"./BaseObject"
                ], function (BaseObject) {
                	"use strict";
                	return BaseObject.extend("be.wl.PersonSkills.model.Skill", {
                		constructor: function (data) {
                			BaseObject.call(this, data);
                			this.Editable = true;
                			this.Deletable = false;
                		},
                		isEmpty:function(){
                			return this.SkillName === "" || this.Score === 0;
                		},
                		isNotEmpty:function(){
                			return this.SkillName !== "" && this.Score >= 0;
                		},
                		getJSON: function () {
                			return {
                				Id: this.Id || 0,
                				PersonId: this.PersonId || 0,
                				SkillName: this.SkillName || "",
                				Score: this.Score || 0
                			};
                		}
                	});
                });
        

    - Person
        
                sap.ui.define([
                	"./BaseObject",
                	"./Skill"
                ], function (BaseObject, Skill) {
                	"use strict";
                	return BaseObject.extend("be.wl.PersonSkills.model.Person", {
                		constructor: function (data) {
                			BaseObject.call(this, data);
                			this.Skills = [];
                			if (data) {
                				this.Birthdate = data.Birthdate;
                				this.setSkills(data.PersonHasSkills.results);
                			}
                		},
                		deleteSkill: function (iIndex) {
                			this.Skills.splice(iIndex, 1);
                		},
                		addEmptySkill: function () {
                			this.Skills.push(new Skill({
                				SkillName: "",
                				Score: ""
                			}));
                		},
                		setSkills: function (aSkills) {
                			this.Skills = aSkills.map((oSkill) => new Skill(oSkill));
                		},
                		getSkills: function () {
                			return this.Skills.filter((oSkill) => oSkill.isNotEmpty()).map((oSkill) => oSkill.getJSON());
                		},
                		getJSON: function () {
                			return {
                				Id: this.Id || 0,
                				Firstname: this.Firstname || "",
                				Lastname: this.Lastname || "",
                				Company: "",
                				Birthdate: this.Birthdate || "",
                				PersonHasSkills: this.getSkills()
                			};
                		}
                	});
                });
        
    - setskills update personstate -> use Person object
    
				"../model/Person"
			    this.data.Person = new Person();
			    this.data.Person = new Person(result.data);
			
	- update view -> remove results
		
		        items="{pers>/Person/Skills}"
- Create
    - Create empty Person object
    
    		createPerson: function () {
    			this.data.Person = new Person();
    			this.data.Person.addEmptySkill();
    			this.data.display = false;
    			this.updateModel();
    		},


    - update detail controller route matched

            if (sObjectId === "NEW") {
            	this.PersonState.createPerson();
            } else {
            	this.getModel().metadataLoaded().then(() => {
            		var sObjectPath = this.getModel().createKey("Persons", {
            			Id: sObjectId
            		});
            		this.getOwnerComponent().oListSelector.selectAListItem("/" + sObjectPath);
            		
            		this.PersonState.getPerson(sObjectId);
            	});
            }

	- add create button to master
	
        	<semantic:addAction>
        		<semantic:AddAction press=".onCreatePerson"/>
        	</semantic:addAction>
	- add create fn to controller
	
        	onCreatePerson:function(oEvent){
        		var bReplace = !Device.system.phone;
        		// set the layout property of FCL control to show two columns
        		this.getOwnerComponent().oListSelector.clearMasterListSelection();
        		this.getModel("appView").setProperty("/layout", "TwoColumnsMidExpanded");
        		this.getRouter().navTo("object", {
        			objectId : "NEW"
        		}, bReplace);
        	},
	
    - add property "display" to state
	- add form for name and skills
	
            xmlns:l="sap.ui.layout" xmlns:f="sap.ui.layout.form"

            <l:VerticalLayout width="100%">
				<f:Form id="generalform" editable="true"  visible="{= !${pers>/display}}">
					<f:layout>
						<f:ResponsiveGridLayout labelSpanXL="3" labelSpanL="3" labelSpanM="4" labelSpanS="12" adjustLabelSpan="false" emptySpanXL="0" emptySpanL="4"
							emptySpanM="0" emptySpanS="0" columnsXL="2" columnsL="1" columnsM="1" singleContainerFullSize="false"/>
					</f:layout>
					<f:formContainers>
						<f:FormContainer title=" ">
							<f:formElements>
								<f:FormElement label="Firstname">
									<f:fields>
										<Input value="{pers>/Person/Firstname}"/>
									</f:fields>
								</f:FormElement>
								<f:FormElement label="Lastname">
									<f:fields>
										<Input value="{pers>/Person/Lastname}"/>
									</f:fields>
								</f:FormElement>
								<f:FormElement label="Birthdate">
									<f:fields>
										<DatePicker
											value="{path: 'pers>/Person/Birthdate', type: 'sap.ui.model.type.Date', formatOptions: { pattern: 'dd/MM/YYYY' , UTC:true}}"/>
									</f:fields>
								</f:FormElement>
							</f:formElements>
						</f:FormContainer>
					</f:formContainers>
				</f:Form>
				<Table visible="{pers>/display}" id="lineItemsList" width="auto" items="{pers>/Person/Skills}"
					noDataText="{i18n>detailLineItemTableNoDataText}" busyIndicatorDelay="{detailView>/lineItemTableDelay}">
					<headerToolbar>
						<Toolbar>
							<Title id="lineItemsTitle" text="{detailView>/lineItemListTitle}" titleStyle="H3" level="H3"/>
						</Toolbar>
					</headerToolbar>
					<columns>
						<Column>
							<Text text="{i18n>detailLineItemTableIDColumn}"/>
						</Column>
						<Column minScreenWidth="Tablet" demandPopin="true" hAlign="End">
							<Text text="{i18n>detailLineItemTableUnitNumberColumn}"/>
						</Column>
					</columns>
					<items>
						<ColumnListItem>
							<cells>
								<ObjectIdentifier title="{pers>SkillName}" text="{Id}"/>
								<ObjectNumber number="{ path: 'pers>Score', formatter: '.formatter.currencyValue' }"/>
							</cells>
						</ColumnListItem>
					</items>
				</Table>
				<Table items="{pers>/Person/Skills}" fixedLayout="false" visible="{= !${pers>/display}}">
					<headerToolbar>
						<Toolbar>
							<Title text="Total: {pers>/Person/Total}" titleStyle="H3" level="H3"/>
						</Toolbar>
					</headerToolbar>
					<columns>
						<Column >
							<Text text="Skill"/>
						</Column>
						<Column >
							<Text text="Score"/>
						</Column>
						<Column >
							<Text text="Edit"/>
						</Column>
						<Column >
							<Text text="Delete"/>
						</Column>
					</columns>
					<items>
						<ColumnListItem>
							<cells>
								<Input value="{pers>SkillName}" editable="{pers>Editable}"/>
								<Input value="{path:'pers>Score',type: 'sap.ui.model.type.Integer'}" editable="{pers>Editable}"/>
								<l:VerticalLayout width="100%">
									<ToggleButton icon="sap-icon://{= ${pers>Editable}?'display':'edit' }"  pressed="{pers>Editable}" />
								</l:VerticalLayout>
								<Button icon="sap-icon://delete" visible="{pers>Deletable}" press=".onDeleteSkill"/>
							</cells>
						</ColumnListItem>
					</items>
				</Table>
			</l:VerticalLayout>	

	- add create function in service en state
        - service
        
                ,
        		createPerson:function(oPerson){
        			var oData = oPerson.getJSON();
        			return this.odata("/Persons").post(oData);
        		}
        - state
            
        		newPerson: function () {
        			return this.PersonService.createPerson(this.data.Person).then((result) => result.data.Id);
        		},

	- add toolbar for save

	        showFooter="true"
	        
    - add save button
    
    		<semantic:footerMainAction >
    			<semantic:FooterMainAction  visible="{= !${pers>/display}}" text="Save" press=".onSave"/>
    		</semantic:footerMainAction>
    		
    - add save to controller
    
    		onSave: function (oEvent) {
    			this.PersonState.newPerson().then((id) => {
    				this.getRouter().navTo("object", {
    					objectId: id
    				}, true);
    			});
    		},
    - delete skill
        - add to basecontroller
        
        		getIndexFromPath: function (oSource) {
        			var sPath = oSource.getBindingContext("pers").getPath();
        			return parseInt(sPath.substr(sPath.lastIndexOf("/") + 1));
        		},
        		
        - add to detail controller
        
        		onDeleteSkill:function(oEvent){
        			this.PersonState.deletePersonSkill(this.getIndexFromPath(oEvent.getSource()));
        		},
        		
        - add to state
            
        		deletePersonSkill: function (iIndex) {
        			this.data.Person.deleteSkill(iIndex);
        			// this.updateModel();
        		}
        
	- on more thing
	    - reactive!
    	    -  UI Changes properties -> only on UI5 changes
    	        -  Skill
    	        
                		SkillNameChanged: function (oEvent) {
                			this.changeEditable();
                		},
                		ScoreChanged: function (oEvent) {
                			this.changeEditable();
                		},
                		changeEditable: function () {
                			this.Editable = !(this.SkillName && this.Score);
                			this.Deletable = !!(this.SkillName || this.Score);
                		},  
    	        - Person
    	        
        	            SkillsChanged: function (oEvent) {
                			if (!this.Skills.some((oSkill) => oSkill.isEmpty())) {
                				this.addEmptySkill();
                			}
                		},
    	    -  Calculated properties - Person total - changes of other values
    	        
                    Object.defineProperty(this, "Total", {
        				get: () => {
        					return (this.getSkills().reduce((iTotal, oSkill) => {
        						iTotal += oSkill.Score
        						return iTotal;
        					}, 0) / this.getSkills().length )|| 0;
        				}
        			});

    	    -  remove refresh model -> observables
    	    
        	        createPerson: function () {
            			// this.Person = new Person();
            			var oPerson = new Person();
            			this.Person = ObservableSlim.create(oPerson, true,  (changes)=>{
            				console.log(JSON.stringify(changes));
            				this.updateModel();
            			});
            			this.Person.addEmptySkill();
            			this.display = false;
            			// this.updateModel();
            		},
