<script>

	/* ### IMPORTS ### */

		// Shared code
		import { setLocalData, getAudits, getRGESN } from './modules/helpers.mjs';
		import { webAppMode } from './modules/webAppMode.mjs';
		import { env } from './modules/env.mjs';
		
	/* ### VARIABLES ### */
	
		let defaultVersion = 'v1';
		let referential; // RGESN content
		let index = 0; // Current audit index identifier
		let audits = [ // Audits progression
			{  
				byCriteria: {},
				byCounters: {
					satisfied: 0,
					rejected: 0,
					notApplicable: 0
				},
				selectedVersion: defaultVersion, // By default.
			}
		];

	/* ### FUNCTIONS ### */

		function exportFile(content, mimetype, filename) {
			let blob = new Blob([content], {type: mimetype});
			let aElm = document.createElement('a');
			aElm.setAttribute('href', window.URL.createObjectURL(blob));
			aElm.setAttribute('download', filename);
			aElm.click();
			aElm.remove();
		}

		function exportJSON() {
			const json = JSON.stringify(audits[index]);
			exportFile(json, 'application/json', 'numecodiag-data.json');
		}

		function exportCSV() {
			let csv = 'ID;Thématique;Libellé du critère;Évaluation;Commentaire;\n';
			for(const criterion of referential.criteres) {
				csv += `${criterion.id};${criterion.thematique};${criterion.critere};`;
				const assessed = audits[index].byCriteria[criterion.id];
				if(assessed !== undefined) {
					switch(assessed.status) {
						case 'satisfied':
							csv += 'conforme;';
								break;
						case 'rejected':
							csv += 'non conforme;';
							break;
						case 'not-applicable':
							csv += 'non applicable;';
							break;
						default:
							csv += 'à évaluer;';
							break;
					}
				}
				else {
					csv += 'à évaluer;';
				}
				if(assessed !== undefined) {
					if(assessed.analysis !== undefined) {
						csv += `"${assessed.analysis}"`;
					}
					else {
						csv += ';';
					}
				}
				else {
					csv += ';';
				}
				csv += ';\n';
			}
			exportFile(csv, 'text/csv', 'numecodiag-results.csv');
		}

		function exportHTML() {
			alert(`Retrouvez votre badge en HTML dans votre dossier téléchargement. Vous pouvez afficher ce badge dans vos communications lorsque 100% des critères sont évalués.`);
			const nbOfCriteria= referential.criteres.length;
			const counters = audits[index].byCounters;
			const assessed = counters.satisfied + counters.rejected + counters.notApplicable;
			const assessedPercent = (assessed/nbOfCriteria * 100).toFixed(0);
			const score = ((counters.satisfied + counters.notApplicable) / nbOfCriteria * 100).toFixed(0);
			const html = `<div class="numecodiag-badge" style="display:flex; align-items: center; gap: 1em; max-width: 60ch; padding: 1em"><img src=" data:image/png;base64,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" alt="Logo MiNum_Eco"><p>Après auto-évaluation, ce service numérique a un taux de conformité de ${score}% pour ${assessedPercent}% des critères évalués selon le <a href="https://ecoresponsable.numerique.gouv.fr/publications/referentiel-general-ecoconception/" target="_blank">Référentiel Général d'Écoconception de Services Numériques</a> (RGESN). </p></div>`;
			exportFile(html, 'text/html', 'numecodiag-badge.html');
		}

		function importJSON(file) {
			const reader = new FileReader();		
			reader.onloadend = e => {
				try {
					const data = JSON.parse(e.target.result);
					if(confirm('Attention : cette action entraîne la perte des données du diagnostic actuel. Continuer ?')) {
						audits[index] = data;
						setLocalData('audits', JSON.stringify(audits));
						alert('Diagnostic chargé !');
					}
				}
				catch(e) {
					alert('Impossible de lire le fichier.')
				}			
			} 
			reader.readAsText(file);
		}

		function resetAudit() {
			if(confirm('Attention : cette action entraîne la perte des données du diagnostic actuel. Continuer ?')) {
				// Udpates runtime values
				audits[index].byCriteria = {};
				audits[index].byCounters.satisfied = 0;
				audits[index].byCounters.rejected = 0;
				audits[index].byCounters.notApplicable = 0;
				audits[index].selectedVersion = defaultVersion;
				// Udpates storage
				setLocalData('audits', JSON.stringify(audits));
				alert('Diagnostic effacé !');
			}
		}

	/* ### PROCEDURAL ### */

		getAudits()
			.then((data) => audits = JSON.parse(data))
			.catch((warning) => console.warn(warning))
			.finally(() => getRGESN(audits[index].selectedVersion)
				.then((rgesn) => referential = rgesn))
				.catch((error) => console.error(error));

		if(webAppMode) {
			window.onstorage = () => 
				getAudits()
					.then((data) => audits = JSON.parse(data))
					.catch((warning) => console.warn(warning));
		}
		else {
			env.storage.onChanged.addListener(() => {
				getAudits()
					.then((data) => audits = JSON.parse(data))
					.catch((warning) => console.warn(warning));  
			});
		}
				
</script>

<main>
	{#if webAppMode}
		<a href="/">Retour</a>
	{/if}
	<h1>NumÉcoDiag | Options</h1>
	<h2>Exporter, communiquer</h2>
	<button on:click="{exportCSV}">Exporter les résultats (CSV)</button>
	<button on:click="{exportJSON}">Exporter les données (JSON)</button>
	<button on:click="{exportHTML}">Télécharger le badge (HTML)</button>
	<form on:submit|preventDefault="{(e) => importJSON(e.target.elements['file'].files[0])}">
		<label for="file">
			<h2>Importer un diagnostic (JSON)</h2>
			<input 
				id="file" 
				name="numecodiag-data" 
				type="file" 
				accept="application/json" 
				required>
		</label>
		<button>Valider</button>
	</form>
	<h2>Configurer</h2>
	<button on:click="{resetAudit}">Réinitialiser le diagnostic</button>
</main>

<style>
	@font-face {
		font-display: swap;
		font-family: "Marianne";
		src: url("../static/marianne-regular.woff2") format("woff2");
	}
	@font-face {
		font-display: swap;
		font-family: "Marianne";
		src: url("../static/marianne-bold.woff2") format("woff2");
		font-weight: bold;
	}
	:global(main) {
		font-family: "Marianne", sans-serif;
	}
	button, input {
		display: block;
		margin-bottom: .5em;
	}
</style>