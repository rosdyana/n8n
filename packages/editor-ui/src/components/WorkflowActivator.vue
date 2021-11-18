<template>
	<div class="workflow-activator">
		<el-switch
			v-if="!disabled"
			v-loading="loading"
			element-loading-spinner="el-icon-loading"
			:value="workflowActive"
			@change="activeChanged"
			:title="workflowActive?'Deactivate Workflow':'Activate Workflow'"
			:disabled="loading"
			:active-color="getActiveColor"
			inactive-color="#8899AA">
		</el-switch>
		<n8n-tooltip v-else placement="bottom">
			<div slot="content">This workflow has no trigger nodes that require activation</div>
			<el-switch
				v-loading="loading"
				element-loading-spinner="el-icon-loading"
				:value="workflowActive"
				@change="activeChanged"
				:title="workflowActive?'Deactivate Workflow':'Activate Workflow'"
				:disabled="true"
				:active-color="getActiveColor"
				inactive-color="#8899AA">
			</el-switch>
		</n8n-tooltip>

		<div class="could-not-be-started" v-if="couldNotBeStarted">
			<n8n-tooltip placement="top">
				<div @click="displayActivationError" slot="content">The workflow is set to be active but could not be started.<br />Click to display error message.</div>
				<font-awesome-icon @click="displayActivationError" icon="exclamation-triangle" />
			</n8n-tooltip>
		</div>
	</div>
</template>

<script lang="ts">

import { externalHooks } from '@/components/mixins/externalHooks';
import { genericHelpers } from '@/components/mixins/genericHelpers';
import { restApi } from '@/components/mixins/restApi';
import { showMessage } from '@/components/mixins/showMessage';
import { workflowHelpers } from '@/components/mixins/workflowHelpers';
import {
	IWorkflowDataUpdate,
	INodeUi,
} from '../Interface';

import mixins from 'vue-typed-mixins';
import { mapGetters } from "vuex";
import {
	INodeTypeDescription,
} from 'n8n-workflow';
import { ElMessageBoxOptions } from 'element-ui/types/message-box';

export default mixins(
	externalHooks,
	genericHelpers,
	restApi,
	showMessage,
	workflowHelpers,
)
	.extend(
		{
			name: 'WorkflowActivator',
			props: [
				'workflowActive',
				'workflowId',
			],
			data () {
				return {
					loading: false,
				};
			},
			computed: {
				...mapGetters({
					dirtyState: "getStateIsDirty",
				}),
				nodesIssuesExist (): boolean {
					return this.$store.getters.nodesIssuesExist;
				},
				isWorkflowActive (): boolean {
					const activeWorkflows = this.$store.getters.getActiveWorkflows;
					return activeWorkflows.includes(this.workflowId);
				},
				couldNotBeStarted (): boolean {
					return this.workflowActive === true && this.isWorkflowActive !== this.workflowActive;
				},
				getActiveColor (): string {
					if (this.couldNotBeStarted === true) {
						return '#ff4949';
					}
					return '#13ce66';
				},
				disabled(): boolean {
					return !this.containsTrigger;
				},
				containsTrigger(): boolean {
					const foundNodes = this.$store.getters.allNodes.map(({ type }: INodeUi) => this.$store.getters.nodeType(type));
					return foundNodes.filter(((type: INodeTypeDescription) => type.group.includes('trigger'))).length > 0;
				},
			},
			methods: {
				async activeChanged (newActiveState: boolean) {
					if (this.workflowId === undefined) {
						this.$showMessage({
							title: 'Problem activating workflow',
							message: 'The workflow did not get saved yet so can not be set active!',
							type: 'error',
						});
						return;
					}

					if (this.nodesIssuesExist === true) {
						this.$showMessage({
							title: 'Problem activating workflow',
							message: 'It is only possible to activate a workflow when all issues on all nodes got resolved!',
							type: 'error',
						});
						return;
					}

					// Set that the active state should be changed
					let data: IWorkflowDataUpdate = {};

					const activeWorkflowId = this.$store.getters.workflowId;
					if (newActiveState === true && this.workflowId === activeWorkflowId) {
						// If the currently active workflow gets activated save the whole
						// workflow. If that would not happen then it could be quite confusing
						// for people because it would activate a different version of the workflow
						// than the one they can currently see.
						if (this.dirtyState) {
							const importConfirm = await this.confirmMessage(`When you activate the workflow all currently unsaved changes of the workflow will be saved.`, 'Activate and save?', 'warning', 'Yes, activate and save!');
							if (importConfirm === false) {
								return;
							}
						}

						// Get the current workflow data that it gets saved together with the activation
						data = await this.getWorkflowDataToSave();
					}

					data.active = newActiveState;

					this.loading = true;

					try {
						await this.restApi().updateWorkflow(this.workflowId, data);
					} catch (error) {
						const newStateName = newActiveState === true ? 'activated' : 'deactivated';
						this.$showError(error, 'Problem', `There was a problem and the workflow could not be ${newStateName}:`);
						this.loading = false;
						return;
					}

					const currentWorkflowId = this.$store.getters.workflowId;
					let activationEventName = 'workflow.activeChange';
					if (currentWorkflowId === this.workflowId) {
						// If the status of the current workflow got changed
						// commit it specifically
						this.$store.commit('setActive', newActiveState);
						activationEventName = 'workflow.activeChangeCurrent';
					}

					if (newActiveState === true) {
						this.$store.commit('setWorkflowActive', this.workflowId);

						// Show activation dialog
						const foundTriggers = this.$store.getters.allNodes
							.map(({ type }: INodeUi) => this.$store.getters.nodeType(type))
							.filter(((type: INodeTypeDescription) => type.group.includes('trigger')));
						let alertTriggerContent = 'Your trigger will now fire production executions automatically.';
						// if multiple triggers
						if (foundTriggers.length > 1) {
							alertTriggerContent = 'Your triggers will now fire production executions automatically.';
						} else {
							const trigger = foundTriggers[0];
							const serviceName = trigger.displayName.replace(/ trigger/i, '');
							//check if webhook
							if (this.$store.getters.currentWorkflowHasWebhookNode) {
								if (trigger.name === 'Webhook') {
									// check if a standard Webhook trigger
									alertTriggerContent = 'You can now make calls to your production webhook URL.';
								} else {
									alertTriggerContent = `Your workflow will now listen for events from ${serviceName}.`;
								}
							} else if (trigger.polling) {
								//check if a polling trigger
								alertTriggerContent = `Your workflow will now check ${serviceName} for events on a regular basis.`;
							} else if (trigger.displayName === 'Cron') {
								// check if a standard Cron trigger
								alertTriggerContent = 'Your cron trigger will now run on the schedule you have defined.';
							} else if (trigger.displayName === 'Interval') {
								// check if a standard Interval trigger
								alertTriggerContent = 'Your interval trigger will now run on the schedule you have defined.';
							}
						}
						const options: ElMessageBoxOptions  = {
							confirmButtonText: 'Got it',
							dangerouslyUseHTMLString: true,
						};
						const alertMessage = '<p><b>These executions will not show up immediately in the editor</b>, but you can see them in the <a>execution list</a>.</p>';
						this.$alert(`${alertTriggerContent} ${alertMessage}`, 'Workflow activated', options);
					} else {
						this.$store.commit('setWorkflowInactive', this.workflowId);
					}

					this.$externalHooks().run(activationEventName, { workflowId: this.workflowId, active: newActiveState });
					this.$telemetry.track('User set workflow active status', { workflow_id: this.workflowId, is_active: newActiveState });

					this.$emit('workflowActiveChanged', { id: this.workflowId, active: newActiveState });
					this.loading = false;
				},
				async displayActivationError () {
					let errorMessage: string;
					try {
						const errorData = await this.restApi().getActivationError(this.workflowId);

						if (errorData === undefined) {
							errorMessage = 'Sorry there was a problem. No error got found to display.';
						} else {
							errorMessage = `The following error occurred on workflow activation:<br /><i>${errorData.error.message}</i>`;
						}
					} catch (error) {
						errorMessage = 'Sorry there was a problem requesting the error';
					}

					this.$showMessage({
						title: 'Problem activating workflow',
						message: errorMessage,
						type: 'warning',
						duration: 0,
					});
				},
			},
		},
	);
</script>

<style scoped>
.workflow-activator {
	display: inline-block;
}

.could-not-be-started {
	display: inline-block;
	color: #ff4949;
	margin-left: 0.5em;
}

::v-deep .el-loading-spinner {
	margin-top: -10px;
}
</style>
