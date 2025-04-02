<script>
    import '@evidence-dev/tailwind/fonts.css';
    import '../app.css';
    import { EvidenceDefaultLayout, Sidebar } from '@evidence-dev/core-components';
    export let data;
</script>

<div class="flex flex-col min-h-screen w-full">
    <!-- Main Content Area with Sidebar and Breadcrumbs Hidden -->
    <EvidenceDefaultLayout {data} hideBreadcrumbs = true hideSidebar = {true} hideTOC= {true} logo = /logo.png neverShowQueries = true >
        <slot slot="content" />
    </EvidenceDefaultLayout>
</div>