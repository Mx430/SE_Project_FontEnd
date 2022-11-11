<template>
    <div style="width: 75%">
        <a-row>
            <p style="text-align:center; font-size:50px">日程安排</p>
        </a-row>
        <a-row>
        <a-card>
            <div class>
                <a-Calendar :fullscreen = "true" onPanelChange={onPanelChange} />
            </div>
        </a-card>
        </a-row>
    </div>
</template>

<script>
export default{
}
</script>

<style>

</style>

<script>
    import _ from 'lodash'
    import inviteDepartmentMember from '../../components/project/inviteDepartmentMember'
    import createDepartment from '../../components/project/createDepartment'
    import {list} from "../../api/department";
    import {del, forbid, list as getMembers, resume} from "../../api/user";
    import pagination from "../../mixins/pagination";
    import {checkResponse, getApiUrl, getAuthorization, getUploadUrl} from "../../assets/js/utils";
    import {notice} from "../../assets/js/notice";
    import {removeMember} from "../../api/departmentMember";
    import {del as deleteDepartment} from "../../api/department";
    import {getStore} from "../../assets/js/storage";

    export default {
        name: "calendar",
        components: {

        },
        mixins: [pagination],
        data() {
            return {
              
            }
        },
        computed: {
            headers() {
                let headers = getAuthorization();
                let organization = getStore('currentOrganization', true);
                if (organization) {
                    headers.organizationCode = organization.code;
                }
                return headers;
            }
        },
        created() {
            this.currentMenu = this.menus[0];
            this.getMembers({key: 0});
            this.getDepartment();
        },
        methods: {
            getDepartment() {
                this.departmentLoading = true;
                list({page: 1, pageSize: 100}).then(res => {
                    let list = [];
                    if (res.data.list) {
                        res.data.list.forEach((v) => {
                            list.push({
                                key: v.code,
                                title: v.name,
                                isLeaf: !v.hasNext,
                                scopedSlots: {icon: 'custom'}
                            })
                        });
                    }
                    this.treeData = list;
                    this.departmentLoading = false;
                });
            },
            getMembers({key} = {}, reload = true) {
                let app = this;
                if (key != undefined) {
                    this.currentDepartmentCode = '';
                    this.currentMenu = this.menus[key];
                    this.selectedKeys = [key.toString()];
                    this.requestData.searchType = key;
                }
                app.loading = true;
                if (reload) {
                    this.pagination.page = 1;
                }
                getMembers(this.requestData).then(res => {
                    if (reload) {
                        app.members = res.data.list;
                    } else {
                        app.members = app.members.concat(res.data.list);
                    }
                    app.pagination.total = res.data.total;
                    app.showLoadingMore = app.pagination.total > app.members.length;
                    app.loading = false;
                    app.loadingMore = false
                });
            },
            search: _.debounce(
                function () {
                    if (!this.keyword) {
                        this.list = [];
                    }
                    if (this.keyword.length <= 1) {
                        return false;
                    }
                    this.requestData.keyword = this.keyword;
                    this.getMembers();
                }, 500
            ),
            onLoadMore() {
                this.loadingMore = true;
                this.pagination.page++;
                this.getMembers({}, false);
            },
            onSelect(selectedKeys, e) {
                // this.onLoadData(e.node);
                this.selectedKeys = [];
                this.currentMenu = e.node.dataRef;
                this.currentDepartmentCode = e.node.dataRef.key;
                this.currentTreeNode = e.node;
                let app = this;
                this.requestData.searchType = 4;
                this.requestData.departmentCode = e.node.dataRef.key;
                app.loading = true;
                getMembers(this.requestData).then(res => {
                    app.members = res.data.list;
                    app.pagination.total = res.data.total;
                    app.showLoadingMore = app.pagination.total > app.members.length;
                    app.loading = false;
                    app.loadingMore = false
                });
            },
            onLoadData(treeNode) {
                return new Promise((resolve) => {
                    list({page: 1, pageSize: 100, pcode: treeNode.dataRef.key}).then(res => {
                        let list = [];
                        if (res.data.list.length) {
                            res.data.list.forEach((v) => {
                                list.push({
                                    key: v.code,
                                    title: v.name,
                                    isLeaf: !v.hasNext,
                                    scopedSlots: {icon: 'custom'}
                                })
                            });
                        }
                        treeNode.dataRef.isLeaf = !list.length > 0;
                        treeNode.dataRef.children = list;
                        this.treeData = [...this.treeData];
                        resolve();
                    });
                })
            },
            createDepartmentSuccess(data) {
                let list = [...this.treeData];
                list.push({
                    key: data.code,
                    title: data.name,
                    isLeaf: true,
                    scopedSlots: {icon: 'custom'}
                });
                this.treeData = [];
                this.treeData = list;
                this.showCreateDepartment = false
            },
            createChildDepartmentSuccess() {
                this.onLoadData(this.currentTreeNode);
                this.showCreateChildDepartment = false;
            },
            editDepartmentSuccess(data) {
                this.currentTreeNode.dataRef.title = data;
                this.showEditDepartment = false;
            },
            deleteDepartment() {
                let app = this;
                this.$confirm({
                    title: '您确定要删除当前部门吗？',
                    content: `删除部门会同时删除其子部门，部门中的成员不会被移出组织。`,
                    okText: '删除',
                    okType: 'danger',
                    cancelText: '再想想',
                    onOk() {
                        deleteDepartment(app.currentDepartmentCode).then((res) => {
                            if (!checkResponse(res)) {
                                return;
                            }
                            if (app.currentTreeNode.$parent.dataRef) {
                                app.onLoadData(app.currentTreeNode.$parent);
                                app.onSelect(null, {node: app.currentTreeNode.$parent});
                            } else {
                                app.getMembers({key: 0});
                                app.getDepartment();
                            }
                            notice({title: '删除成功'}, 'notice', 'success');
                        });
                        return Promise.resolve();
                    }
                });
            },
            forbidAccount(member, index) {
                let app = this;
                this.$confirm({
                    title: '您确定要停用当前帐号吗？',
                    content: `被停用的帐号将无法访问该${this.actionTitle}，但帐号信息仍保留，方便工作交接和管理。支持帐号恢复`,
                    okText: '停用',
                    okType: 'danger',
                    cancelText: '再想想',
                    onOk() {
                        forbid(member.code).then((res) => {
                            if (!checkResponse(res)) {
                                return;
                            }
                            app.members.splice(index, 1);
                            notice({title: '已成功停用账号'}, 'notice', 'success');
                        });
                        return Promise.resolve();
                    }
                });
            },
            resumeAccount(member, index) {
                let app = this;
                this.$confirm({
                    title: '您确定要启用账号吗？',
                    content: `被启用的帐号将重新加入该${this.actionTitle}`,
                    okText: '启用',
                    okType: 'primary',
                    cancelText: '再想想',
                    onOk() {
                        resume(member.code).then((res) => {
                            if (!checkResponse(res)) {
                                return;
                            }
                            app.members.splice(index, 1);
                            notice({title: '已成功启用账号'}, 'notice', 'success');
                        });
                        return Promise.resolve();
                    }
                });
            },
            deleteAccount(member, index) {
                let app = this;
                this.$confirm({
                    title: `确认从${this.actionTitle}中移除成员吗？`,
                    content: `移除后该账号在${this.actionTitle}内的相关信息将会被清除`,
                    okText: '移除',
                    okType: 'danger',
                    cancelText: '再想想',
                    onOk() {
                        if (app.currentDepartmentCode) {
                            removeMember(member.code, app.currentDepartmentCode).then((res) => {
                                if (!checkResponse(res)) {
                                    return;
                                }
                                app.members.splice(index, 1);
                                notice({title: '移除成功'}, 'notice', 'success');
                            });
                        } else {
                            del(member.code).then((res) => {
                                if (!checkResponse(res)) {
                                    return;
                                }
                                app.members.splice(index, 1);
                                notice({title: '移除成功'}, 'notice', 'success');
                            });
                        }
                        return Promise.resolve();
                    }
                });
            },
            emitEmpty() {
                this.$refs.keywordInput.focus();
                this.keyword = '';
                this.requestData.keyword = '';
                this.getMembers();
            },
            handleChange(info) {
                if (info.file.status === 'uploading') {
                    notice(`正在导入，请稍后...`, 'message', 'loading', 0);
                    this.uploadLoading = true;
                    return
                }
                if (info.file.status === 'done') {
                    console.log(info);
                    this.uploadLoading = false;
                    if (checkResponse(info.file.response, true)) {
                        const count = info.file.response.data;
                        notice(`导入成功`, 'message', 'success');
                        this.getMembers();
                    }
                }
            },
            beforeUpload(file) {
                const isLt2M = file.size / 1024 / 1024 < 2;
                if (!isLt2M) {
                    this.$message.error('文件不能超过2MB!')
                }
                return isLt2M
            },
        }
    }
</script>