<template>
    <v-card flat>
        <v-card-title>Copy a Location</v-card-title>
        <v-card-text>
            <v-text-field 
                v-model="agencyCode" 
                :rules="[rules.required]"
                label="Agency Code">
            </v-text-field>
            <v-text-field 
                v-model="siteNumber" 
                :rules="[rules.required]"
                label="Site Number">
            </v-text-field>
        </v-card-text>
        <v-card-actions>
            <v-btn color="primary" @click="exportLocation">Copy</v-btn>
            <v-tooltip top>
                <template v-slot:activator="{ on }">
                    <v-btn color="primary" v-on="on" icon class="mx-2">
                        <v-icon>mdi-help-circle</v-icon>
                    </v-btn>
                </template>
                <span>Enter an agency code and site number for an existing site that you'd like to copy. The site will be copied to all NWIS hosts as a result.</span>
            </v-tooltip>
        </v-card-actions>
        <v-dialog 
            hide-overlay width="300"
            v-model="loading" 
            >
            <v-card>
                <v-card-text>
                    Processing your request
                    <v-progress-linear 
                        indeterminate 
                        class="mb-0"
                    ></v-progress-linear>
                </v-card-text>
            </v-card>
        </v-dialog>
    </v-card>
</template>

<script>
import _ from "lodash";
import LegacyLocationApi from "@/services/api/LegacyLocationApi.js";

export default {
    name: "CopyLocationCard",

    data() {
        return {
            agencyCode: "",
            siteNumber: "",
            rules: {
                    required: value => !!value || 'Required'
            },
            loading: false,
        }
    },
    methods: {
        exportLocation() {
            this.loading = true;
            LegacyLocationApi.postExport(this.agencyCode, this.siteNumber)
                .then(response => {
                    this.handleExportWorkflowError(response);
                })
                .catch(error => {
                    this.handleExportWorkflowError(error.response)
                })
                .finally(() => {
                        this.loading = false;
                });
        },
        handleExportWorkflowError(response) {
            var workflowFailureMsg = {};
            workflowFailureMsg.exportWorkflowErrors = {
                name: "Export Workflow Errors",
                errors: []
            };
            _.forEach(response.data.workflowSteps, function(w) {
                if (w.name === "Complete Export Workflow") {
                    if (w.success === false) {
                        workflowFailureMsg.exportWorkflowErrors.errors.push({
                            name: w.name,
                            message:
                                "Failed: " + this.parseErrorMessage(w.details)
                        });
                    }
                }
            }.bind(this));
            this.$emit("exportWorkflow", "exportReport", response.data, workflowFailureMsg);
        },
        parseErrorMessage(message){
            if (message.includes("error_message")){
                return JSON.parse(message).error_message;
            } else {
                return message;
            }
        }
    }
};
</script>