(function () {
    'use strict';

    angular.module('EcatApp')
        .controller('Newnewselectortrl', Newnewselectortrl);

    Newnewselectortrl.$inject = ['$state', '$scope', '$modal',
        'dispatcher', 'growl', '$translate', 'Constants',
        'Utils', 'LoginDataStore', 'ShoppingListDataStore', 'ShoppingListManagerDataStore',
        'SessionManager', 'AddToShpppingListUtils'];

    function Newnewselectortrl($state, $scope, $modal,
        dispatcher, growl, $translate, Constants,
        Utils, LoginDataStore, ShoppingListDataStore, ShoppingListManagerDataStore,
        SessionManager, AddToShpppingListUtils) {

        var newselector = this;
        var buttonDisabled = true;
        newselector.allShoppingList = [];
        newselector.defaultEntryName = '';
        newselector.dmsShoppingList = {};
        newselector.subscriptions = [];
        newselector.fromconversion = false;
        newselector.shoppingList = [];
        newselector.shoppingListSelectedItem = null;
        newselector.shoppingListSelectedItemId = 0;
        newselector.select2Options = {
            allowClear: true,
            formatResult: formatOption,
            formatSelection: formatOption,
            escapeMarkup: function (m) {
                return m;
            },

            minimumResultsForSearch: -1
        };
        newselector.showDropdownType = 'all';
        newselector.selectShoppingList = selectShoppingList;
        newselector.addToShoppingListClickBtn = addToShoppingListClickBtn;
        newselector.buttonDisabled = buttonDisabled;
        init();

        function init() {
            newselector.showDropdownType = newselector.showtype || 'all';

            $translate('shoppingList.new')
                .then(function (defaultEntryName) {
                    newselector.defaultEntryName = defaultEntryName;
                });

            registerSubscription();

            getShoppingList();

            $scope.$on('$destroy', destroy);

        }

        function addSubmitHandler(entry) {
            newselector.dmsShoppingList = entry.shoppingList;
            if (entry.signature == newselector.signature) {
                addToShoppingListClickBtn();
            }
        }

        function addToShoppingListClickBtn() {

            var selectedShoppingListItem =
                Utils.getObjectFromSessionStorage('UserDefaultShoppingList');

            console.log('added to shoppingList');

            if (selectedShoppingListItem.id < 0) {
                console.log('adding new ShoppingList:open new modal');
                openAddListModal();
            } else {
                dispatcher.publish('ShoppingListSelector:configrationParams', {
                    publishShoppingList: selectedShoppingListItem,
                    signature: newselector.signature
                });

            }

        }

        function addToWorkerOrder(data) {
            var selectedShoppingListItem =
                Utils.getObjectFromSessionStorage('UserDefaultShoppingList');

            if (data.signature == newselector.signature) {
                switch (data.dataType) {
                    case 'part':
                        AddToShpppingListUtils.addToPart(data.data)
                            .then(function (result) {
                                if (result == true) {
                                    dispatcher.publish(
                                        'ShoppingListSelector:addToWorkOrderSuccess', {
                                            publishShoppingList: selectedShoppingListItem,
                                            signature: newselector.signature
                                        });
                                }
                            });

                        break;
                    case 'servicemenu':
                        AddToShpppingListUtils.addToServiceMenu(data.data)
                            .then(function (result) { });

                        break;
                    case 'labour':
                        AddToShpppingListUtils.addToLabour(data.data)
                            .then(function (result) {
                                if (result == true) {
                                    dispatcher.publish(
                                        'ShoppingListSelector:addToWorkOrderSuccess', {
                                            publishShoppingList: selectedShoppingListItem,
                                            signature: newselector.signature
                                        });
                                }
                            });

                        break;
                }
            }
        }

        function changeButtonActiveHandler(data) {
            if (newselector.showaddbutton != false &&
                data.signature == newselector.signature) {
                newselector.buttonDisabled = data.disabled;
            }
        }

        function destroy() {

            newselector.subscriptions
                .map(function (item) {
                    item.unsubscribe();
                });
            newselector.subscriptions.length = 0;
            newselector.subscriptions = null;
        }

        function formatOption(data) {
            var slItem = getShoppingListItemById(data.id);
            if (slItem.shoppingListId == '-1') {
                return data.text;
            } else {
                var faClass = '';
                if (slItem.orderTypeId == 1) {
                    if (slItem.isEheckList) {
                        faClass = 'fa-shoppinglist-echeck';
                    }
                } else {
                    faClass = 'fa-shoppinglist-workorder';
                }

                return '<i class="fa fa-shopping-cart ' +
                    faClass + '"></i>   ' +
                    slItem.listName + ' - ' +
                    slItem.statusString;
            }

        }

        function getShoppingListItemById(shoppingListId) {
            return _.find(newselector.shoppingList, {
                shoppingListId: shoppingListId
            });
        }

        function getShoppingList() {
            dispatcher
                .publishAndWaitForReply('shoppingList:getWorkOrderAndShoppingList', null)
                .then(
                function (data) {
                    newselector.allShoppingList = data.shoppingListDropDownList;
                    var slData = data.shoppingListDropDownList;

                    var showSlOrWorker = newselector.showtype;
                    if (showSlOrWorker == Constants.SL_SELECT_SHOWSHOPPINGLIST) {
                        slData = _.filter(data.shoppingListDropDownList, function (o) {
                            return typeof o.appUserId !== 'undefined';
                        });
                    }

                    if (showSlOrWorker == Constants.SL_SELECT_SHOWWORKORDER) {
                        slData = _.filter(data.shoppingListDropDownList, function (o) {
                            return typeof o.appUserId == 'undefined';
                        });
                    }

                    if (showSlOrWorker == Constants.SL_SELECT_SHOWSHOPPINGLIST_NOLOCK) {
                        slData = _.filter(data.shoppingListDropDownList, function (o) {
                            return typeof o.appUserId !== 'undefined' && o.locked == false;
                        });
                    }

                    slData.forEach(function (item, index) {
                        item.statusString = showStatus(item);
                    });

                    newselector.shoppingList = slData;
                    handleUserShoppingList(slData);
                },

                function (err) {
                });

            //dispatcher.publish('ShoppingListView:getShoppingList');
        }

        function getShoppingListFromSession() {
            return Utils.getObjectFromSessionStorage('UserDefaultShoppingList') || {};
        }

        function getActiveShoppingListFromSession() {
            return SessionManager.getSessionData('activeShoppingListDetails');
        }

        function handleUserShoppingList(data) {

            newselector.shoppingList = [];

            var defaultEntry = {
                shoppingListId: '-1',
                listName: newselector.defaultEntryName
            };

            // if (!newselector.fromconversion) {
            //     newselector.shoppingList.push(defaultEntry);
            // }

            newselector.shoppingList.push(defaultEntry);
            for (var i = 0; i < data.length; i++) {
                newselector.shoppingList.push(data[i]);
            }

            var sessionShoppingList = null;

            if (newselector.showtype == Constants.SL_SELECT_SHOWSHOPPINGLIST_NOLOCK) {
                sessionShoppingList = getActiveShoppingListFromSession();
            } else {
                sessionShoppingList = getShoppingListFromSession();
            }

            var slName = '';
            if (!_.isEmpty(sessionShoppingList)) {
                newselector.shoppingListSelectedItemId = sessionShoppingList.id || sessionShoppingList.shoppingListId;
                slName = sessionShoppingList.name || sessionShoppingList.listName;
            } else {
                newselector.shoppingListSelectedItemId =
                    newselector.shoppingList[0].shoppingListId;
                slName = newselector.shoppingList[0].listName;
            }

            setSessionSoppingList({
                id: newselector.shoppingListSelectedItemId,
                name: slName
            });

        }

        function openAddListModal() {

            var modalOptions = {
                backdrop: true,
                backdropClick: false,
                dialogFade: false,
                keyboard: true,
                windowClass: 'modal-primary',
                templateUrl: 'components/addToShoppingList/addNewShoppingListModal.html',
                controller: 'AddNewShoppingListModal',
                controllerAs: 'newmodalc',
                resolve: {
                    dmsShoppingList: function () {
                        return newselector.dmsShoppingList;
                    }
                }
            };
            var modalInstance = $modal.open(modalOptions);
            modalInstance.result
                .then(
                function (result) {

                    setSessionSoppingList({
                        id: result.shoppingListId,
                        name: result.listName
                    });
                    if (typeof newselector.nochangeactive == 'undefined' ||
                        newselector.nochangeactive == false) {
                        SessionManager.setSessionData(result, 'activeShoppingListDetails');
                        SessionManager
                            .setSessionData(result.shoppingListId, 'activeShoppingListId');
                    }

                    dispatcher
                        .publish('ShoppingList:refreshSelectShoppingList', newselector.signature);
                    var publishShoppingList = {
                        id: result.shoppingListId,
                        name: result.listName
                    };
                    dispatcher.publish('ShoppingListSelector:configrationParams', {
                        publishShoppingList: publishShoppingList,
                        signature: newselector.signature
                    });
                },

                function () {
                    console.log('ShoppingList has not been added.');
                });

        }

        function registerSubscription() {
            var subscriptionSelectedItemsEmptyUpdate = dispatcher
                .subscribe('ShoppingList:selectedItemsEmptyUpdate', changeButtonActiveHandler);
            var subscriptionAddSubmitHandler = dispatcher
                .subscribe('ShoppingListSelector:onAddSubmit', addSubmitHandler);
            var subscriptionRefreshShoppingList = dispatcher.subscribe(
                'ShoppingList:refreshSelectShoppingList', refreshShoppingList);
            var subscriptionAddToWorkerOrder = dispatcher
                .subscribe('ShoppingList:addToWorkerOrder', addToWorkerOrder);
            newselector.subscriptions.push(subscriptionAddToWorkerOrder);
            newselector.subscriptions.push(subscriptionSelectedItemsEmptyUpdate);
            newselector.subscriptions.push(subscriptionAddSubmitHandler);
            newselector.subscriptions.push(subscriptionRefreshShoppingList);
        }

        function refreshShoppingList(signature) {
            if (newselector.showdropdown != false &&
                signature == newselector.signature) {
                getShoppingList();
            }
        }

        function selectShoppingList() {
            var selectItem = getShoppingListItemById(newselector.shoppingListSelectedItemId);
            setSessionSoppingList({
                id: selectItem.shoppingListId,
                name: selectItem.listName
            });

            if (typeof newselector.nochangeactive == 'undefined' ||
                newselector.nochangeactive == false) {
                SessionManager.setSessionData(selectItem, 'activeShoppingListDetails');
                SessionManager.setSessionData(selectItem.shoppingListId, 'activeShoppingListId');
                newselector.shoppingList.forEach(function (item, index) {
                    item.statusString = showStatus(item);
                });
            }

            dispatcher.publish('ShoppingList:refreshSelectShoppingList', newselector.signature);

            dispatcher.publish('ShoppingListSelector:changeSelect', {
                publishShoppingList: selectItem,
                signature: newselector.signature
            });
        }

        function setSessionSoppingList(sessionShoppingList) {
            Utils.setObjectInSessionStorage('UserDefaultShoppingList', sessionShoppingList);
        }

        function showStatus(shoppingList) {
            return ShoppingListManagerDataStore.showStatus(shoppingList);
        }

    }

})();
