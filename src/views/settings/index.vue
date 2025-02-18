<template>
    <div
        class="settings-container"
        :class="[{dark: theme === 'dark'}]"
    >
        <div class="s-row">
            <p class="s-title">{{local('Setting')}}</p>
        </div>
        <div class="scroll-view">
            <div class="s-item-block">
                <p class="s-item-title">{{local('Source')}}</p>
                <fv-button
                    :theme="theme"
                    icon="OneDriveAdd"
                    style="width: 150px;"
                    @click="addSource"
                >{{local('Add New Source')}}</fv-button>
                <fv-list-view
                    :value="dbList"
                    :theme="theme"
                    style="width: 100%; height: auto; margin-top: 15px;"
                    @chooseItem="switchDSDB($event.item)"
                >
                    <template v-slot:listItem="x">
                        <div class="list-view-item">
                            <i class="ms-Icon ms-Icon--Link"></i>
                            <p class="item-name">{{x.item.name}}</p>
                            <fv-button
                                v-show="x.item.status == 502 || SourceIndexDisabled(x.index)"
                                :theme="theme"
                                class="control-btn"
                                background="rgba(255, 200, 0, 1)"
                                :title="local(`Can't find data_structure.json on this source, shall we init new one ?`)"
                                @click="showInitDS(x.index)"
                            >
                                <i class="ms-Icon ms-Icon--EraseTool"></i>
                            </fv-button>
                            <fv-button
                                :theme="theme"
                                class="control-btn"
                                @click="removeDS(x.item)"
                            >
                                <i class="ms-Icon ms-Icon--RemoveLink"></i>
                            </fv-button>
                        </div>
                    </template>
                </fv-list-view>
            </div>
            <div class="s-item-block">
                <p class="s-item-title">{{local('Theme')}}</p>
                <fv-button
                    :theme="theme"
                    fontSize="16"
                    borderRadius="50"
                    style="width: 40px; height: 40px;"
                    :title="theme === 'light' ? `${local('Switch to the dark theme')}` : `${local('Switch to the light theme')}`"
                    @click="toggleTheme(v)"
                >
                    <i
                        class="ms-Icon"
                        :class="[`ms-Icon--${theme === 'light' ? 'Sunny' : 'ClearNight'}`]"
                    ></i>
                </fv-button>
            </div>
            <div class="s-item-block">
                <p class="s-item-title">{{local('Language')}}</p>
                <fv-Combobox
                    v-model="cur_language"
                    :theme="theme"
                    :options="languages"
                    placeholder="Choose A Language"
                    :background="theme === 'dark' ? 'rgba(36, 36, 36, 1)' : ''"
                    @choose-item="chooseLanguage"
                ></fv-Combobox>
            </div>
        </div>
        <init-ds
            :show.sync="show.initDS"
            :theme="theme"
            :db_index="db_index"
        ></init-ds>
    </div>
</template>

<script>
import { mapMutations, mapState, mapGetters } from "vuex";
import initDs from "@/components/settings/initDs.vue";
const { dialog } = require("electron").remote;

export default {
    components: {
        initDs,
    },
    data() {
        return {
            cur_language: {},
            languages: [
                { key: "en", text: "English" },
                { key: "cn", text: "简体中文" },
            ],
            dbList: [],
            db_index: -1,
            show: {
                initDS: false,
            },
            lock: {
                switchDSDB: true,
            },
        };
    },
    watch: {
        $route() {
            this.languageInit();
            this.dataSourceSync();
        },
        language() {
            this.languageInit();
        },
        "show.initDS"() {
            this.dataSourceSync();
        },
    },
    computed: {
        ...mapState({
            init_status: (state) => state.init_status,
            data_index: (state) => state.data_index,
            data_path: (state) => state.data_path,
            ds_db_list: (state) => state.ds_db_list,
            language: (state) => state.language,
            theme: (state) => state.theme,
        }),
        ...mapGetters(["local", "ds_db"]),
        v() {
            return this;
        },
        SourceIndexDisabled() {
            return (index) => {
                if (!this.ds_db_list[index]) return true;
                let id = this.ds_db_list[index].get("id").write();
                return id === null;
            };
        },
    },
    mounted() {
        this.languageInit();
        this.dataSourceSync();
    },
    methods: {
        ...mapMutations({
            reviseConfig: "reviseConfig",
            reviseDS: "reviseDS",
            reviseData: "reviseData",
            toggleTheme: "toggleTheme",
            syncDS: "syncDS"
        }),
        languageInit() {
            this.cur_language = this.languages.find(
                (item) => item.key === this.language
            );
        },
        chooseLanguage(item) {
            this.reviseConfig({
                v: this,
                language: item.key,
            });
        },
        dataSourceSync() {
            // 此函数初始化数据源的DB //
            // 同时也会初始化ListView列表项目 //
            let pathList = this.data_path;
            let db_array_result = this.$load_ds_file(pathList);
            if (db_array_result.status == 404 && !this.init_status) {
                this.$barWarning(
                    this.local(
                        "There is no source, please add a data source to getting started."
                    ),
                    {
                        status: "warning",
                        autoClose: -1,
                    }
                );
                return;
            }
            let db_array = db_array_result.db_array;
            let dbList = [];
            let ds_db_list = [];
            db_array.forEach((el, idx) => {
                dbList.push({
                    key: idx,
                    name: pathList[idx],
                    path: pathList[idx],
                    choosen: idx === this.data_index,
                    status: el.status,
                    msg: el.msg,
                    disabled: () => el.status === 500 || !this.lock.switchDSDB,
                    db: el.db,
                });
                ds_db_list.push(el.db);
            });
            this.dbList = dbList;
            this.reviseData({
                ds_db_list: ds_db_list,
            });
        },
        async addSource() {
            let path = (
                await dialog.showOpenDialog({
                    properties: ["openDirectory"],
                })
            ).filePaths[0];
            if (!path) return;
            let pathList = this.data_path;
            if (!pathList.find((url) => url === path)) pathList.push(path);
            await this.reviseConfig({
                v: this,
                data_path: pathList,
            });
            this.dataSourceSync();
            let index = pathList.indexOf(path);
            if (!this.SourceIndexDisabled(index)) {
                if (this.data_index === index)
                    await this.reviseConfig({
                        v: this,
                        data_index: -1,
                    });
                this.reviseConfig({
                    v: this,
                    data_index: index,
                });
            }
        },
        switchDSDB(item) {
            this.lock.switchDSDB = false;
            let index = this.data_path.indexOf(item.path);
            this.reviseConfig({
                v: this,
                data_index: index,
            });
            this.lock.switchDSDB = true;
        },
        showInitDS(db_index) {
            this.db_index = db_index;
            this.show.initDS = true;
        },
        removeDS(db_item) {
            this.$infoBox(
                this.local("Are you sure to remove this data source?"),
                {
                    status: "error",
                    title: this.local("Remove Data Source"),
                    confirmTitle: this.local("Confirm"),
                    cancelTitle: this.local("Cancel"),
                    theme: this.theme,
                    confirm: () => {
                        let url = db_item.path;
                        let index = this.data_path.indexOf(url);
                        this.data_path.splice(index, 1);
                        this.dataSourceSync();
                    },
                    cancel: () => {},
                }
            );
        },
    },
};
</script>

<style lang="scss">
.settings-container {
    position: relative;
    width: 100%;
    height: 100%;
    background: whitesmoke;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    transition: all 0.3s;

    &.dark {
        background: rgba(36, 36, 36, 1);

        .s-title {
            color: whitesmoke;
        }

        .scroll-view {
            .s-item-block {
                .s-item-title {
                    color: whitesmoke;
                }
            }
        }
    }

    .s-row {
        position: relative;
        margin: 25px 0px;
        padding: 0px 15px;
        box-sizing: border-box;
        display: flex;
        align-items: center;
    }

    .s-title {
        font-size: 24px;
        user-select: none;
        cursor: default;
    }

    .scroll-view {
        position: relative;
        width: 100%;
        flex: 1;
        display: flex;
        flex-direction: column;
        align-items: center;
        overflow: auto;

        .s-item-block {
            position: relative;
            width: calc(100% - 30px);
            height: auto;
            line-height: 2.5;
            display: flex;
            flex-direction: column;

            .s-item-title {
                user-select: none;
                cursor: default;
            }

            .list-view-item {
                position: relative;
                width: 100%;
                display: flex;
                align-items: center;

                .item-name {
                    margin-left: 15px;
                    font-size: 13px;
                    flex: 1;
                }

                .control-btn {
                    width: 40px;
                    height: 40px;
                    margin-right: 5px;
                }
            }
        }
    }
}
</style>