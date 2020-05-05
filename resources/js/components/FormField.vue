<template>
	<div v-if="dependenciesSatisfied">
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
	import {
		FormField,
		HandlesValidationErrors
	} from 'laravel-nova'

	export default {
		mixins: [FormField, HandlesValidationErrors],

		props: ['resourceName', 'resourceId', 'field'],

		mounted() {
			this.registerDependencyWatchers(this.$root, function () {
				this.updateDependencyStatus();
			});
		},

		data() {
			return {
				dependencyValues: {},
				dependenciesSatisfied: false,
			}
		},

		methods: {
			addWatcher(component, attribute, value) {
				if (attribute === 'selectedResource') {
					value = (value && value.value) || null;
				}

				if (Array.isArray(value)) {
					this.dependencyValues[component.field.attribute] = []

					value.forEach((object, key) => {
						this.dependencyValues[component.field.attribute].push(object[attribute]);
					})
				} else {
					this.dependencyValues[component.field.attribute] = value;
				}
			},

			// @todo: refactor entire watcher procedure, this approach isn't maintainable ..
			registerDependencyWatchers(root, callback) {
				callback = callback || null

				root.$children.forEach(component => {
					if (this.componentIsDependency(component)) {
						// @todo: change `findWatchableComponentAttribute` to return initial state(s) of current dependency.
						let attribute = this.findWatchableComponentAttribute(component),
								initialValue = component.field.value; // @note: quick-fix for issue #88

						component.$watch(attribute, (value) => {
							this.addWatcher(component, attribute, value)
							this.updateDependencyStatus()
						}, { immediate: true })

						// @todo: move to initial state
						// @note quick-fix for issue #88
						if (attribute === 'fieldTypeName') {
							initialValue = component.field.resourceLabel
						}

						// @todo: replace with `updateDependencyStatus(initial_value)` and let it resolve dependency state
						this.dependencyValues[component.field.attribute] = initialValue
					}

					this.registerDependencyWatchers(component)
				});

				if (callback !== null) {
					callback.call(this)
				}
			},

			// @todo: not maintainable, move to factory
			findWatchableComponentAttribute(component) {
				let attribute;

				switch(component.field.component) {
					case 'belongs-to-many-field':
					case 'belongs-to-field':
						attribute = 'selectedResource';
						break;
					case 'morph-to-field':
						attribute = 'fieldTypeName';
						break;
					default:
						attribute = 'value';
				}

				return attribute;
			},

			componentIsDependency(component) {
				if(component.field === undefined) {
					return false;
				}

				for (let dependency of this.field.dependencies) {
					// #93 compatability with flexible-content, which adds a generated attribute for each field
					if(component.field.attribute === (this.field.attribute + dependency.field)) {
						return true;
					}
				}

				return false;
			},

			// @todo: align this method with the responsibility of updating the dependency, not verifying the dependency "values"
			updateDependencyStatus() {
				for (let dependency of this.field.dependencies) {
					// #93 compatability with flexible-content, which adds a generated attribute for each field
					let selectedValue = this.dependencyValues[(this.field.attribute + dependency.field)];

					if (dependency.hasOwnProperty('empty') && !selectedValue) {
						this.dependenciesSatisfied = true;
						return;
					}

					if (dependency.hasOwnProperty('notEmpty') && selectedValue) {
						this.dependenciesSatisfied = true;
						return;
					}

					if (dependency.hasOwnProperty('nullOrZero') && 1 < [undefined, null, 0, '0'].indexOf(selectedValue) ) {
						this.dependenciesSatisfied = true;
						return;
					}

					if (dependency.hasOwnProperty('value')) {
						let result = false

						if (Array.isArray(selectedValue) && selectedValue.includes(dependency.value)) {
							result = true
						} else if (dependency.value == selectedValue) {
							result = true
						}

						this.dependenciesSatisfied = result;

						return;
					}
				}

				this.dependenciesSatisfied = false;
			},

			fill(formData) {
				if(this.dependenciesSatisfied) {
					_.each(this.field.fields, field => {
						if (field.fill) {
							field.fill(formData)
						}
					})
				}
			}

		}
	}
</script>
