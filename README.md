...
"{"catalogueCode":3,"orderSourceType":1,"priceTypeId":"0","programId":null,"regionId":null,"serviceMenuItems":[{"componentCode":"12030277","parts":[],"labours":[],"fluids":[]}],"fluidItems":[{"fluidNumber":"1847947","fluidCode":84,"containerCode":"P","quantity":1}],"activeShoppingListId":"123"}"

dispatcher.publish('ShoppingList:addToWorkerOrder',
                            {
                                signature: data.signature,
                                dataType: 'servicemenu',
                                data: serviceMenuData
                            });


showAddSuccess
