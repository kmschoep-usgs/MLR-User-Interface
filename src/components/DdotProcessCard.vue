<template>
    <v-card flat>
        <v-card-title>Upload and Process a Ddot file</v-card-title>
        <v-card-subtitle>When creating or modifying sites outside your Center, please coordinate with the other Center's Local Data Manager to ensure continuity of ownership</v-card-subtitle>
        <v-card-text>
            <v-form>
                <v-file-input v-model="ddotFile" label="Select Ddot File to Upload"></v-file-input>
            </v-form>
        </v-card-text>
        <v-card-actions>
            <v-btn class="mx-2" color="primary" @click="validateButtonClicked">Validate</v-btn>
            <v-btn color="primary" @click="uploadButtonClicked">Validate and Update records</v-btn>
        </v-card-actions>
    </v-card>
</template>

<script>
import DdotApi from "@/services/api/DdotApi.js";
import { EventBus } from "@/components/EventBus.js";

export default {
    name: "DdotProcessCard",

    data() {
        return {
            ddotFile: null,
            districtCodes: null,
            transactionTypes: null
        };
    },
    methods: {
        validateButtonClicked() {
            this.parseDdotFile().then(() => {
                this.validateDdotFile();
            });
        },
        uploadButtonClicked() {
            this.parseDdotFile().then(() => {
                this.uploadDdotFile();
            });
        },
        parseDdotFile() {
            return DdotApi.parseDdot(this.ddotFile);
        },
        uploadDdotFile() {
            DdotApi.uploadDdot(this.ddotFile)
                .then(response => {
                    this.emitSnackbarUpdate(response);
                    this.workflowErrors(response);
                })
                .catch(error => {
                    this.emitSnackbarUpdate(error);
                });
        },
        validateDdotFile() {
            DdotApi.validateDdot(this.ddotFile)
                .then(response => {
                    this.emitSnackbarUpdate(response);
                    this.workflowErrors(response);
                })
                .catch(error => {
                    this.emitSnackbarUpdate(error);
                });
        },
        emitSnackbarUpdate(response) {
            EventBus.$emit("snackbar-update", response);
        },
        workflowErrors(response) {
            var workflowFailureMsg = {};
            var workflowErrors = null;
            if (response.data.workflowSteps.length > 0) {
                var workflowFailure = response.data.workflowSteps.filter(
                    function(x) {
                        return x.name.search("workflow") > 0;
                    }
                );
                if (workflowFailure.length > 0) {
                    workflowFailureMsg.workflowStatus = {
                        name: workflowFailure[0].name,
                        message:
                            " (" +
                            JSON.parse(workflowFailure[0].details)
                                .error_message +
                            ") : No Transactions were processed. Error details listed below: "
                    };
                }

                workflowErrors = response.data.workflowSteps.filter(function(
                    x
                ) {
                    return x.name.search("workflow") < 0;
                });
            }

            if (workflowFailureMsg.workflowStatus === undefined) {
                workflowFailureMsg.workflowStatus = {
                    name: "Status",
                    message:
                        response.data.numberSiteSuccess +
                        " Transactions Succeeded, " +
                        response.data.numberSiteFailure +
                        " Transactions Failed"
                };
            }

            if (workflowErrors != null) {
                workflowFailureMsg.workflowLevelErrors = {
                    name: "Workflow-level Errors",
                    errors: []
                };
                workflowErrors.forEach(function(e) {
                    workflowFailureMsg.workflowLevelErrors.errors.push({
                        name: e.name,
                        message: JSON.parse(e.details).error_message
                    });
                });
            }

            if (response.data.sites.length > 0) {
                workflowFailureMsg.siteErrors = {
                    name: "Site-level Errors and Warnings",
                    errors: this.parseSiteErrorRows(response.data.sites)
                };
            }
            this.$emit("validate-workflow", response.data, workflowFailureMsg);
        },
        parseSiteErrorRows(errorList) {
            var result = [];
            errorList.forEach(function(s) {
                var site = s.agencyCode.trim() + "-" + s.siteNumber.trim();
                var steps = s.steps;
                steps.forEach(function(st) {
                    var detailMsg = JSON.parse(st.details);
                    if (detailMsg.hasOwnProperty("error_message")) {
                        if (typeof detailMsg.error_message === "object") {
                            Object.keys(detailMsg.error_message).forEach(
                                function(key) {
                                    result.push({
                                        name: site,
                                        message:
                                            st.name +
                                            " Fatal Error: " +
                                            key +
                                            " - " +
                                            detailMsg.error_message[key]
                                    });
                                }
                            );
                        } else {
                            result.push({
                                name: site,
                                message:
                                    st.name +
                                    " Fatal Error: " +
                                    detailMsg.error_message
                            });
                        }
                    }
                    if (detailMsg.hasOwnProperty("validator_message")) {
                        if (
                            detailMsg.validator_message.hasOwnProperty(
                                "fatal_error_message"
                            )
                        ) {
                            Object.keys(
                                detailMsg.validator_message.fatal_error_message
                            ).forEach(function(key) {
                                result.push({
                                    name: site,
                                    message:
                                        st.name +
                                        " Fatal Error: " +
                                        key +
                                        " - " +
                                        detailMsg.validator_message
                                            .fatal_error_message[key]
                                });
                            });
                        }
                        if (
                            detailMsg.validator_message.hasOwnProperty(
                                "warning_message"
                            )
                        ) {
                            Object.keys(
                                detailMsg.validator_message.warning_message
                            ).forEach(function(key) {
                                result.push({
                                    name: site,
                                    message:
                                        st.name +
                                        " Warning: " +
                                        key +
                                        " - " +
                                        detailMsg.validator_message
                                            .warning_message[key]
                                });
                            });
                        }
                    }
                });
            });
            return result;
        }
    }
};
</script>