# employee-search-
Renders the employee information
/**
 * @description Employee search view to provide employee list
 * @author Rishav Goyal
 */

;
( function () {

    app.views.employeeSearchItem = Backbone.Marionette.ItemView.extend( {
        template: '#employeeSearchView',
        url:{
            //sample : 'https://api.zinghr.com/mobilev6/route/Home/SearchGetEmployeeDirectory'
        },
        events: {
            'click .email_compose_h': 'composeEmail',
            'click .call_user_h': 'callSelectedUser'
        },
        composeEmail: function ( e ) {
            if ( App.settings.isDeviceReady && device.platform == "Android" ) {
                console.log("device is ready");
                var contactEmail = this.model.get( 'Email' );
                cordova.InAppBrowser.open( "mailto:" + contactEmail, "_system", 'enableViewportScale=yes' );
            }
            else if ( App.settings.isDeviceReady && device.platform == "iOS" ) {
                console.log("ios device is ready");
                var contactEmail = this.model.get( 'Email' );
                if ( App.settings.isDeviceReady ) {
                    window.plugin.email.open( {
                        to: [ contactEmail ],
                        body: '',
                        isHtml: true
                    } );
                }
            }

        },
        callSelectedUser: function ( e ) {
            var contactPhone = this.model.get( 'MobileNo' );
            if ( App.settings.isDeviceReady ) {
                cordova.InAppBrowser.open( "tel:" + contactPhone, "_system", 'enableViewportScale=yes' );
            }
        },
        initialize: function () {
            if ( this.model.attributes.FirstName ) {
                this.model.attributes.successCode = 1;
            }
            else if ( this.model.attributes.successCode == 0 ) {
                this.model.attributes.successCode = 0;
            }
        },
        onRender: function () {},
        onClose: function () {
            //console.log("VIEW CLOSE");
        }
    } );

    app.views.employeeSearch = Backbone.Marionette.CollectionView.extend( {
        itemView: app.views.employeeSearchItem,
        events: {
            
        },

        initialize: function ( options ) {
            //console.log("Employee search collection intialized");
            var _this = this;
            //can instantiate with employeeSearch or employeeDirectory
            this.collection = new app.collections.employeeSearch;
            

            var searchdText = null;
            //if searchText contains anything
            if ( options && options.searchText ) {
                searchdText = options.searchText;
                App.collections.employeeSearch = this.collection;
                this.searchEmployees(searchdText);
                
            }else{
                if(App.collections.employeeDirectory && App.collections.employeeDirectory.length){
                    this.collection.reset(App.collections.employeeDirectory);

                }else{
                    App.sessionVars.employeeData = {
                    options: this.options,
                    pageIndex: "1"
                    };
                    App.collections.employeeDirectory = this.collection;
                    this.getEmployeeDirectory();
                }
                
            }
 
        },

        onRender: function () {},

        onClose: function () {},

        getEmployeeDirectory: function(){
            var requestData = {
                "TransType":"SearchGetEmployeeDirectory",
                "PageSize":"10",
                "PageIndex":App.sessionVars.employeeData.pageIndex,
                "SearchText":"",
                "MobileAppVersion":"72"
            };
            App.controller.showSecondPopup( "loader" );
                    App.communication.POST( {
                        //here we are mentioning the data
                        //requesting data through POST
                        collection: this.collection,
                        url: App.settings[ App.settings.production_server + "_server_new" ] + App.settings.urls_new.employee_search,
                        dataType: "json",
                        data: JSON.stringify( requestData ),
                        contentType: "application/json; charset=utf-8",
                        successCallback: function ( data ) {
                             var existingIndex = parseInt(App.sessionVars.employeeData.pageIndex);
                            App.events.stopLoadMoreSpinner();
                            //console.log(data)
                            App.events.closeSecondPopups();
                            if ( data.Code == 1 ) {
                                //console.log("Employee Search success");
                                App.sessionVars.allowAttributes = null;
                                if ( data && data.DOConfig && data.DOConfig.Response && data.DOConfig.Response[ 0 ] && data.DOConfig.Response[ 0 ][ 0 ] ) {
                                    App.sessionVars.allowAttributes = data.DOConfig.Response[ 0 ][ 0 ];
                                    App.sessionVars.allowAttributes.MobileApplicable = App.sessionVars.allowAttributes.MobileApplicable.toLocaleLowerCase() == 'true';
                                    App.sessionVars.allowAttributes.EmailApplicable = App.sessionVars.allowAttributes.EmailApplicable.toLocaleLowerCase() == 'true'
                                }
                                if ( !App.sessionVars.allowAttributes ) {
                                    App.sessionVars.allowAttributes = {};
                                    App.sessionVars.allowAttributes.EmailApplicable = true;
                                    App.sessionVars.allowAttributes.MobileApplicable = true;
                                }
                                if ( data && data.DO && data.DO.Response ) {
                                    var parsedData = data.DO.Response[ 0 ];
                                    App.sessionVars.attributes = [];
                                    if ( data.DO.Response[ 1 ] ) {
                                        _.each( data.DO.Response[ 1 ], function ( value, keys ) {
                                            App.sessionVars.attributes.push( value.AttributeTypeDescription );
                                        } );
                                    }
                                    if ( parsedData && parsedData.length > 0 ) {
                                        App.sessionVars.employeeData.pageIndex = (++existingIndex).toString();
                                        this.collection.reset( parsedData );
                                    }
                                    else {
                                        this.collection.reset( {
                                            'successCode': 0
                                        } );
                                    }
                                }
                            }
                            else if ( data.Code == 0 ) {
                                App.controller.showSecondPopup( "alert", {
                                    title: "No data fetch", //TODO: Change it and show no data in template
                                    message: data.Message,
                                    callBack: function () {
                                        App.events.closeSecondPopups();
                                    }
                                }, null );
                            }

                        },//here is the code that renders data without searching it
                        onCompleteCallBack: function () {

                        },
                        errorCallback: App.controller.ajaxErrorCall
                    } );
        
        },

              /*      onCompleteCallBack: function() {
                        App.events.stopLoadMoreSpinner();
                        // App.events.closeSecondPopups();
                    },
                    errorCallback: function(data) {
                        App.events.stopLoadMoreSpinner();
                        //console.log("Error in load more.");
                    }
                });
            } else {
                //console.log("Load more getting net error");
            }

        }*/






        searchEmployees : function(searchdText){
            var requestData = {
                "TransType": "SearchGetEmployeeDirectory",
                "PageSize": "10",
                "PageIndex": App.sessionVars.approvalsData.pageIndex,
                "SearchText": searchdText,
                "MobileAppVersion": App.settings.applicationVersion
            };
            if ( ( searchdText ) && ( searchdText.replace( /\s/g, "" ).length > 0 ) ) {
                if ( navigator && navigator.onLine ) {
                    App.controller.showSecondPopup( "loader" );
                    App.communication.POST( {
                        collection: this.collection,
                        url: App.settings[ App.settings.production_server + "_server_new" ] + App.settings.urls_new.employee_search,
                        dataType: "json",
                        data: JSON.stringify( requestData ),
                        contentType: "application/json; charset=utf-8",
                        successCallback: function ( data ) {
                            //console.log(data)
                            App.events.closeSecondPopups();
                            if ( data.Code == 1 ) {
                                //console.log("Employee Search success");
                                App.sessionVars.allowAttributes = null;
                                if ( data && data.DOConfig && data.DOConfig.Response && data.DOConfig.Response[ 0 ] && data.DOConfig.Response[ 0 ][ 0 ] ) {
                                    App.sessionVars.approvalsData.pageIndex = (++existingIndex).toString();
                                    App.sessionVars.allowAttributes = data.DOConfig.Response[ 0 ][ 0 ];
                                    App.sessionVars.allowAttributes.MobileApplicable = App.sessionVars.allowAttributes.MobileApplicable.toLocaleLowerCase() == 'true';
                                    App.sessionVars.allowAttributes.EmailApplicable = App.sessionVars.allowAttributes.EmailApplicable.toLocaleLowerCase() == 'true'
                                }
                                if ( !App.sessionVars.allowAttributes ) {
                                    App.sessionVars.approvalsData.pageIndex = (++existingIndex).toString();
                                    App.sessionVars.allowAttributes = {};
                                    App.sessionVars.allowAttributes.EmailApplicable = true;
                                    App.sessionVars.allowAttributes.MobileApplicable = true;
                                }
                                if ( data && data.DO && data.DO.Response ) {
                                    App.sessionVars.approvalsData.pageIndex = (++existingIndex).toString();
                                    var parsedData = data.DO.Response[ 0 ];
                                    App.sessionVars.attributes = [];
                                    if ( data.DO.Response[ 1 ] ) {
                                        App.sessionVars.approvalsData.pageIndex = (++existingIndex).toString();
                                        _.each( data.DO.Response[ 1 ], function ( value, keys ) {
                                            App.sessionVars.attributes.push( value.AttributeTypeDescription );
                                        } );
                                    }
                                    if ( parsedData && parsedData.length > 0 ) {
                                        App.sessionVars.approvalsData.pageIndex = (++existingIndex).toString();
                                        App.collections.employeeSearch.reset( parsedData );
                                    }
                                    else {
                                        App.collections.employeeSearch.reset( {
                                            'successCode': 0
                                        } );
                                    }
                                }
                            }
                            else if ( data.Code == 0 ) {
                                App.controller.showSecondPopup( "alert", {
                                    title: "No data fetch", //TODO: Change it and show no data in template
                                    message: data.Message,
                                    callBack: function () {
                                        App.events.closeSecondPopups();
                                    }
                                }, null );
                            }

                        },
                        onCompleteCallBack: function () {

                        },
                        errorCallback: App.controller.ajaxErrorCall
                    } );

                }
                else {
                    App.controller.showSecondPopup( "alert", {
                        title: App.settings.messageTitle.ohSnapTitle,
                        message: App.settings.messageSubject.networkError,
                        callBack: function () {
                            App.events.closeSecondPopups();
                        }
                    }, null );
                }
            }
            else {
                this.collection.reset( {
                    'successCode': 2
                } );
            }
        },
//not an important function
         loadMoreEmployeeData: function(e) {
            var scrollTop = App.pages.employeeSearch.mainContainer.scrollTop
            var scrollHeight = App.pages.employeeSearch.mainContainer.scrollHeight - App.pages.employeeSearch.mainContainer.clientHeight;
            var scrollWeight = scrollTop /scrollHeight ;
            
            // if((window.scrollY + $(window).height()) > App.settings.deviceHeight && (window.scrollY + $(window).height()) >= ($('#bodyRegion').height())) {
            if(scrollWeight>=1){
                if(App.sessionVars.loadMoreOn){
                    return false;
                }
                App.sessionVars.loadMoreOn = true;
                App.sessionVars.oasPageNumber = App.sessionVars.oasPageNumber + 1;
                App.pages.oas.$el.find('.load_more_spinner_h').show();
                App.controller.unbindScrollDetect();
                setTimeout(function(){
                    App.vent.on("stopLoadMoreSpinner", function() {
                        App.controller.stopLoadMoreSpinner();
                    });
                    App.views.employeeSearch.loadMoreEmployeeData();
            },400);
            }
        }

    } );
    
    

} )();
