<script lang='ts'>
    import AutoComplete from 'simple-svelte-autocomplete'
    import { getContext, onDestroy } from 'svelte';

    export let field;
    export let label;
    export let validation;
    export let placeholder;
    export let dataSourceType;
    export let searchEvent;
    export let apiUrl;
    export let queryParam;
    export let staticOptions;
    export let dataProvider;
    export let loading;
    export let results;
    export let labelFieldName;
    export let valueFieldName;

    let resultsPromise;
    let loadingResolver;

    const { styleable } = getContext('sdk');
    const component = getContext('component');
    const formContext = getContext('form');
    const formStepContext = getContext('form-step');
    const fieldGroupContext = getContext('field-group');

    let fieldState;
    let fieldApi;
    let errors;
    let content;
    let unsubscribe;
    let text = "";
    let selectedItem;
    let initialItemsPromise;

    const formApi = formContext?.formApi;
    const labelPos = fieldGroupContext?.labelPosition || 'above';
    $: formStep = formStepContext ? $formStepContext || 1 : 1;
    $: labelClass = labelPos === 'above' ? '' : `spectrum-FieldLabel--${labelPos}`;


    $: dataContext = {
        text,
        value: selectedItem?.[valueFieldName]
    };

    $: delay = dataSourceType === 'static' ? 0 : 100;

    $: if (formApi && field) {
        const formField = formApi.registerField(
            field,
            'text',
            '', ///TODO: Default value?
            false,
            validation,
            formStep
        );

        unsubscribe = formField?.subscribe(async (fieldValue) => {
            fieldState = fieldValue?.fieldState;
            fieldApi = fieldValue?.fieldApi;

            const value = fieldState?.value;

            if (value && !initialItemsPromise) {
                console.log('Loading initial items');
                placeholder = 'Loading';
                initialItemsPromise = await getItems(value);
                const item = (await initialItemsPromise).find((item) => item[valueFieldName] === value);

                if (item) {
                    selectedItem = item;
                }
                else {
                    selectedItem = {
                        [valueFieldName]: value,
                        [labelFieldName]: value
                    };
                }

                placeholder = '';
            }

        });
    };

    $: if (dataSourceType === 'query') {
        const parsedLoading = loading.toLowerCase() === 'true' || loading === '1';

        if (parsedLoading && !loadingResolver) {
            console.log('Loading new results from query');
            resultsPromise = new Promise((resolve) => {
                loadingResolver = resolve;
            });
        }
        else if (!parsedLoading && loadingResolver) {
            console.log('Got results from query');
            loadingResolver(JSON.parse(results));
            loadingResolver = null;
        }
    };

    $: if (dataSourceType === 'static') {
        labelFieldName = 'label';
        valueFieldName = 'value';
    };

    onDestroy(() => {
        fieldApi?.deregister();
        unsubscribe?.();
    });

    async function getItems(keyword: string) {
        switch (dataSourceType) {
            case 'budibase':
                return dataSource?.rows;
            case 'query':
                if (searchEvent) {
                    searchEvent({keyword});
                }
                await new Promise((resolve) => setTimeout(resolve, 100));
                return await resultsPromise
            case 'static':
                return staticOptions;
            case 'api':
                const response = await fetch(`${apiUrl}?${queryParam}=${keyword}`);
                return await response.json();
        }
    }

    function changeHandler() {
        if (selectedItem) {
            fieldApi?.setValue(selectedItem[valueFieldName]);
        }
    }

</script>


<div class='spectrum-Form-item' use:styleable={$component.styles}>
    {#if !formContext}
        <div class='placeholder'>Form components need to be wrapped in a form</div>
    {:else}
        <label
            class:hidden={!label}
            for={fieldState?.fieldId}
            class={`spectrum-FieldLabel spectrum-FieldLabel--sizeM spectrum-Form-itemLabel ${labelClass}`}
        >
          {label || ' '}
        </label>
        <div class='spectrum-Form-itemField'>
            <AutoComplete searchFunction="{getItems}" bind:selectedItem bind:text {placeholder} {labelFieldName} {valueFieldName} onChange="{changeHandler}" {delay} />
        </div>
        {#if fieldState?.error}
            <div class='error'>{fieldState.error}</div>
        {/if}
    {/if}
</div>

<style>
  .placeholder {
    color: var(--spectrum-global-color-gray-600);
  }
  label {
    white-space: nowrap;
  }
  label.hidden {
    padding: 0;
  }
    .spectrum-Form-itemField {
    position: relative;
    width: 100%;
  }
  .spectrum-FieldLabel--right,
  .spectrum-FieldLabel--left {
    padding-right: var(--spectrum-global-dimension-size-200);
  }
</style>