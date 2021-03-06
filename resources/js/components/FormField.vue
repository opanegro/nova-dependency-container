<template>
	<div v-if="dependenciesSatisfied" :class="hasSize === true ? 'w-1/2' : ''">
		<div v-for="childField in field.fields">
			<component
				:is="'form-' + childField.component"
                :errors="errors"
				:resource-id="resourceId"
				:resource-name="resourceName"
				:field="childField"
				:ref="'field-' + childField.attribute"
			/>
		</div>
	</div>
</template>

<script>
	import {FormField, HandlesValidationErrors} from 'laravel-nova'

	export default {
		mixins: [FormField, HandlesValidationErrors],

		props: ['resourceName', 'resourceId', 'field'],

		mounted() {
            this.checkHasSize()
			this.registerDependencyWatchers(this.$root)
			this.updateDependencyStatus()
		},

		data() {
			return {
				dependencyValues: {},
				dependenciesSatisfied: false,
                hasSize: false,
			}
		},

        created() {
		    Nova.$on('nova-dependency-container-' + this.field.attribute, this.dependencyChange)
        },

		methods: {

			registerDependencyWatchers(root) {
				root.$children.forEach(component => {
					if (this.componentIsDependency(component)) {

						component.$watch('value', (value) => {
                            this.dependencyValues[component.field.attribute] = (typeof value === 'object' && value !== null ? value.value : value);
                            this.updateDependencyStatus()
						}, {immediate: true})

						this.dependencyValues[component.field.attribute] = component.field.value;

					}

					this.registerDependencyWatchers(component)
				})
			},

			componentIsDependency(component) {
				if(component.field === undefined) {
					return false;
				}

				for (let dependency of this.field.dependencies) {
					if(component.field.attribute === dependency.field) {
						return true;
					}
				}

				return false;
			},

			updateDependencyStatus() {
				for (let dependency of this.field.dependencies) {
					if(dependency.hasOwnProperty('notEmpty') && ! this.dependencyValues[dependency.field]) {
						this.dependenciesSatisfied = false;
						return;
					}

                    if(dependency.hasOwnProperty('empty') && this.dependencyValues[dependency.field]) {
                        this.dependenciesSatisfied = false;
                        return;
                    }

					if(dependency.hasOwnProperty('value') && this.dependencyValues[dependency.field] != dependency.value) {
						this.dependenciesSatisfied = false;
						return;
					}

                    if(dependency.hasOwnProperty('falseValue') && this.dependencyValues[dependency.field] == dependency.falseValue) {
                        this.dependenciesSatisfied = false;
                        return;
                    }
				}

				this.dependenciesSatisfied = true;
			},

			fill(formData) {
				if(this.dependenciesSatisfied) {
					_.each(this.field.fields, field => {
						if (field.fill) {
							field.fill(formData)
						}
					})
				}
			},

			checkHasSize() {
                if (_.isArray(this.field.fields)) {
                    let checkSize = this.field.fields.filter((data) => {
                        return typeof data.size !== 'undefined'
                    })
                    if (checkSize.length > 0) {
                        this.hasSize = true
                    }
                }
            }

		}
	}
</script>
