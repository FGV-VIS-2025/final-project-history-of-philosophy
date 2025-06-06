<svelte:head>
    <title>Philosophy Timeline</title>
</svelte:head>

<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Overlock:ital,wght@0,400;0,700;0,900;1,400;1,700;1,900&display=swap" rel="stylesheet">

<script>
    import { onMount, afterUpdate } from 'svelte';
    import * as d3 from 'd3';
    import './timeline.css';
    import './infos.js';
    import categorias from './tooltip_categories.json';
    // import { filosofos } from './infos.js';
    import filosofos from '../../components/data/philosophers2.json';
    import { philosophers } from '../../components/data/philosophers.json';
    import { getAlivePhilosophersIdx, getPhilosopherDesc } from './philosophers_manipulation';
    import {base} from '$app/paths';
    import glossary from '../../components/data/glossary.json';

    // glossary processing function
    // and exclusion of terms already wrapped in <b> or <em> tags
    function processTextWithGlossary(text, glossaryData) {
        if (!text || !glossaryData) return text;
        
        let processedText = text;
        
        // Sort glossary terms by length (longest first) to avoid partial matches
        // This is crucial for compound terms like "modal logic" vs "logic"
        const sortedTerms = Object.keys(glossaryData).sort((a, b) => b.length - a.length);
        
        sortedTerms.forEach(term => {
            // Escape special regex characters
            const escapedTerm = term.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
            
            // For compound words, we want to match the exact phrase
            // Use word boundaries for both single and multi-word terms
            const regex = new RegExp(`\\b${escapedTerm}\\b`, 'gi');
            
            // Only replace if not already wrapped in a glossary-term span, <b>, or <em> tags
            processedText = processedText.replace(regex, (match, offset, string) => {
                // Check if this match is already inside a glossary-term span
                const beforeMatch = string.substring(0, offset);
                const afterMatch = string.substring(offset + match.length);
                
                // Count opening and closing spans before this match
                const openSpans = (beforeMatch.match(/<span[^>]*class="[^"]*glossary-term[^"]*"/g) || []).length;
                const closeSpans = (beforeMatch.match(/<\/span>/g) || []).length;
                
                // If we're inside a span, don't replace
                if (openSpans > closeSpans) {
                    return match;
                }
                
                // Check if the term is inside <b> or <em> tags
                // Look for the closest opening tag before our match
                const tagsBefore = beforeMatch.match(/<\/?(?:b|em|strong|i)\b[^>]*>/gi) || [];
                const tagsAfter = afterMatch.match(/<\/?(?:b|em|strong|i)\b[^>]*>/gi) || [];
                
                // Track which tags are currently open
                let openTags = [];
                tagsBefore.forEach(tag => {
                    const isClosing = tag.startsWith('</');
                    const tagName = tag.match(/<\/?(\w+)/)[1].toLowerCase();
                    
                    if (isClosing) {
                        // Remove the tag from openTags if it's being closed
                        openTags = openTags.filter(t => t !== tagName);
                    } else {
                        // Add the tag to openTags if it's being opened
                        openTags.push(tagName);
                    }
                });
                
                // Check if we're currently inside b, em, strong, or i tags
                const isInsideFormattingTag = openTags.some(tag => 
                    ['b', 'em', 'strong', 'i'].includes(tag)
                );
                
                // If we're inside formatting tags, don't create a glossary tooltip
                if (isInsideFormattingTag) {
                    return match;
                }
                
                return `<span class="glossary-term" data-definition="${glossaryData[term].replace(/"/g, '&quot;')}">${match}</span>`;
            });
        });
        
        return processedText;
    }


    filosofos.sort((a, b) => a.nascimento - b.nascimento);
    let selectedFilosofo = null;
    let selectedFilosofoInfo = null;
    let isSplitView = false;    
    let containerWidth = 0;  
    
    
    // tooltip state variables
    let tooltipVisible = false;
    let tooltipContent = '';
    let tooltipX = 0;
    let tooltipY = 0;
                            
    let initialYear = -600; 
    let finalYear = 2000;   
    let stepYears = 100;    

    const anos = d3.range(initialYear, finalYear + 1, stepYears);

    const colors = {
        background: '#f5efe6',
        timeline: '#6b4f3a',
        accent: '#8c7a6d',
        text: '#3e2d23',
        highlight: '#8C7351'
    };

    // Background Elements
    const eras = [
        { name: 'Antiga', start: -600, end: 500 },
        { name: 'Medieval', start: 500, end: 1500 },
        { name: 'Renascentista', start: 1500, end: 1600 },
        { name: 'Moderna', start: 1600, end: 1800 },
        { name: 'Contemporânea', start: 1800, end: 2000 }
    ];
    const colorsBackground = ['#ffcccc', '#ccffcc', '#ccccff', '#ffffcc', '#ffccff'];

    // Posição das x categorias para serem usadas no template  
    let categoriaPositions = [];

    // tooltip functions
    function showTooltip(event) {
            const target = event.target;
            if (target.classList.contains('glossary-term')) {
                tooltipContent = target.dataset.definition;
                tooltipVisible = true;
                
                // Use setTimeout to ensure the tooltip is rendered before positioning
                setTimeout(() => {
                    positionTooltip(event);
                }, 0);
            }
        }

        function hideTooltip(event) {
            // Only hide if we're not moving to another glossary term
            if (!event.relatedTarget || !event.relatedTarget.classList.contains('glossary-term')) {
                tooltipVisible = false;
            }
        }

        function moveTooltip(event) {
            if (tooltipVisible) {
                positionTooltip(event);
            }
        }

        function positionTooltip(event) {
        const tooltipElement = document.querySelector('.glossary-tooltip');
        if (!tooltipElement) return;

        const mouseX = event.clientX;
        const mouseY = event.clientY;
        const offset = 10; // Distance from cursor
        
        // Get viewport dimensions
        const viewportWidth = window.innerWidth;
        const viewportHeight = window.innerHeight;
        
        // Get tooltip dimensions
        const tooltipRect = tooltipElement.getBoundingClientRect();
        const tooltipWidth = tooltipRect.width;
        const tooltipHeight = tooltipRect.height;
        
        // Calculate available space in each direction
        const spaceRight = viewportWidth - mouseX;
        const spaceLeft = mouseX;
        const spaceBottom = viewportHeight - mouseY;
        const spaceTop = mouseY;
        
        // Determine horizontal position
        let x;
        if (spaceRight >= tooltipWidth + offset) {
            // Enough space on the right
            x = mouseX + offset;
        } else if (spaceLeft >= tooltipWidth + offset) {
            // Not enough space on right, but enough on left
            x = mouseX - tooltipWidth - offset;
        } else {
            // Not enough space on either side, choose the side with more space
            if (spaceRight > spaceLeft) {
                x = mouseX + offset;
            } else {
                x = mouseX - tooltipWidth - offset;
            }
            
            // Ensure tooltip doesn't go off-screen
            x = Math.max(5, Math.min(x, viewportWidth - tooltipWidth - 5));
        }
        
        // Determine vertical position
        let y;
        if (spaceBottom >= tooltipHeight + offset) {
            // Enough space below
            y = mouseY + offset;
        } else if (spaceTop >= tooltipHeight + offset) {
            // Not enough space below, but enough above
            y = mouseY - tooltipHeight - offset;
        } else {
            // Not enough space above or below, choose the side with more space
            if (spaceBottom > spaceTop) {
                y = mouseY + offset;
            } else {
                y = mouseY - tooltipHeight - offset;
            }
            
            // Ensure tooltip doesn't go off-screen
            y = Math.max(5, Math.min(y, viewportHeight - tooltipHeight - 5));
        }
        
        // Apply the calculated position
        tooltipX = x;
        tooltipY = y;
    }

    function selectFilosofo(filosofo) {
        /*Função para selecionar filósofo e ativar split view*/
        d3.selectAll(`.category-connection.${selectedFilosofo?.nome.replace(/\s+/g, '-')}`)
            .style('opacity', 0);
        if (isSplitView && selectedFilosofo === filosofo) {
            closeDetailView();
        } else {
            selectedFilosofo = filosofo;
            isSplitView = true;
            selectedFilosofoInfo = getPhilosopherDesc(selectedFilosofo.nome);
            updateCategoryConnections(selectedFilosofo.nome); // Atualiza conexões
            d3.selectAll(`.category-connection.${selectedFilosofo.nome.replace(/\s+/g, '-')}`)
                .style('opacity', 0.6);
        }
    }

    function closeDetailView() {
        if (selectedFilosofo) {
            updateCategoryConnections(selectedFilosofo.nome); // Esconde conexões
        }
        selectedFilosofo = null;
        isSplitView = false;
        selectedFilosofoInfo = null;
        tooltipVisible = false; // Hide tooltip when closing detail view
    }


    function handleResize() {
        drawTimeLine();
    }

    function updateCategoryConnections(filosofoNome) {
        const shouldShow = selectedFilosofo?.nome === filosofoNome;
        d3.selectAll(`.category-connection.${filosofoNome.replace(/\s+/g, '-')}`)
            .style('opacity', shouldShow ? 0.6 : 0);
    }

    function showCategoryConnections(filosofoNome) {
        if (selectedFilosofo?.nome !== filosofoNome) {
            d3.selectAll(`.category-connection.${filosofoNome.replace(/\s+/g, '-')}`)
                .style('opacity', 0.6);
        }
    }

    function hideCategoryConnections(filosofoNome) {
        if (selectedFilosofo?.nome !== filosofoNome) {
            d3.selectAll(`.category-connection.${filosofoNome.replace(/\s+/g, '-')}`)
                .style('opacity', 0);
        }
    }

    function drawTimeLine() {
        const svg = d3.select('#timeline');
        svg.selectAll("*").remove();
        
        const timelineContainer = document.getElementById('timeline-container');
        if (!timelineContainer) return;

        containerWidth = timelineContainer.clientWidth;
        const height = 6000;
        const margin = { top: 80, right: containerWidth / 12, bottom: 80, left: containerWidth / 12 };

        const y = d3.scaleLinear()
            .domain([initialYear, finalYear])
            .range([margin.top, height - margin.bottom]);

        const linearGradient = svg.append('linearGradient')
            .attr('id', 'bg-gradient')
            .attr('x1', '0%')
            .attr('y1', '0%')
            .attr('x2', '0%')  
            .attr('y2', '100%');

        // For each era, add two stops to the gradient
        eras.forEach((era, i) => {
            const startOffset = y(era.start) / height;
            const endOffset = y(era.end) / height;

            const color = colorsBackground[i % colorsBackground.length];

            linearGradient.append('stop')
                .attr('offset', startOffset)
                .attr('stop-color', color);

            linearGradient.append('stop')
                .attr('offset', endOffset)
                .attr('stop-color', color);
        });

        // Background
        svg.append('rect')
            .attr('x', 0)
            .attr('y', 0)
            .attr('width', containerWidth)
            .attr('height', height)
            .attr('fill', 'url(#bg-gradient)');

        // Largura de cada coluna com base no número de categorias
        const columnWidth = (containerWidth - margin.left - margin.right) / categorias.length;

        // Delimitações verticais entre as colunas 
        const delimitations = d3.range(0, categorias.length + 1).map(i => margin.left + i * columnWidth);

        // Posição central x de cada categoria no topo da timeline
        categoriaPositions = categorias.map((cat, i) => ({
            ...cat,
            pos: (delimitations[i] + delimitations[i + 1]) / 2
        }));

        // Rótulos dos anos
        svg.selectAll('.year-label')
            .data(anos)
            .enter()
            .append('text')
            .attr('class', 'year-label')
            .attr('x', margin.left - margin.left / 3)
            .attr('y', d => y(d))
            .attr('dy', '0.35em')
            .attr('text-anchor', 'end')
            .style('opacity', 0)
            .style('opacity', 1)
            .text(d => d);

        // Linhas horizontais
        svg.selectAll('.year-line')
            .data(anos)
            .enter()
            .append('line')
            .attr('class', 'year-line')
            .attr('x1', margin.left )
            .attr('x2', containerWidth - margin.right / 3)
            .attr('y1', d => y(d))
            .attr('y2', d => y(d))
            .attr('stroke', colors.accent)
            .attr('stroke-width', 0.5)
            .style('opacity', 0.4);

        // Posições x de cada filósofo
        function getXForPosition(position){
            return margin.left + position/12 * (containerWidth - margin.left - margin.right)
        }
        let filosofosProcessados = []
        let xPosFilos = [];
        filosofos.forEach(filosofo => {
            const alive = getAlivePhilosophersIdx(filosofosProcessados, filosofo.nascimento);
            // const alivePositions = alive.map(index => xPosFilos[index]).sort((a, b) => a - b);
            const alivePositions = xPosFilos.filter((_, index) => alive.includes(index)).sort((a, b) => a - b);
            let val = alive.length + 1
            for(let i = 0; i < alivePositions.length; i++){
                const possiblePosition = getXForPosition(i + 1);
                if(possiblePosition < alivePositions[i]){
                    val = i + 1;
                    break;
                }
            }
            xPosFilos.push(getXForPosition(val));
            filosofosProcessados.push(filosofo);
        });

        // Adição do filósofo na timeline
        svg.selectAll('.filosofo-line')
            .data(filosofos)
            .enter()
            .append('line')
            .attr('class', 'filosofo-line')
            .attr('data-filosofo', d => d.nome)
            .attr('x1', (_, i) => xPosFilos[i])
            .attr('x2', (_, i) => xPosFilos[i])
            .attr('y1', d => y(d.nascimento))
            .attr('y2', d => y(d.morte))
            .attr('stroke', colors.timeline)
            .attr('stroke-width', 2.5)
            .attr('stroke-linecap', 'round')
            .style('cursor', 'pointer') 
            .on('click', (event, d) => selectFilosofo(d))
            .attr('class', d => `filosofo-line ${selectedFilosofo?.nome === d.nome ? 'selected' : ''}`);

        // Conexão categorias aos filósofos
        filosofos.forEach((filosofo, i) => {
            const xFilosofo = xPosFilos[i];
            const yNascimento = y(filosofo.nascimento);
            
            filosofo.categorias.forEach(categoriaNome => {
                const categoria = categoriaPositions.find(c => c.nome === categoriaNome);
                if (categoria) {
                    svg.append('line')
                        .attr('class', `category-connection ${filosofo.nome.replace(/\s+/g, '-')}`)
                        .attr('x1', categoria.pos)
                        .attr('x2', xFilosofo)
                        .attr('y2', yNascimento)
                        .attr('stroke', colors.highlight)
                        .attr('stroke-width', 1.3)
                        .attr('stroke-dasharray', '4 2')
                        .style('opacity', selectedFilosofo?.nome === filosofo.nome ? 0.6 : 0); 

                    svg.append('line')
                        .attr('class', `selected-connection ${filosofo.nome.replace(/\s+/g, '-')}`)
                        .attr('x1', categoria.pos)
                        .attr('x2', xFilosofo)
                        .attr('y2', yNascimento)
                        .attr('stroke', colors.highlight)
                        .attr('stroke-width', 1)
                        .style('opacity', 0); 
                }
            });
        });

        // Label do filósofo
        filosofos.forEach((filosofo, i) => {
            const padding = 3;
            const fontSize = 14;
            const nome = filosofo.nome;
            const areaHeight = y(filosofo.morte) - y(filosofo.nascimento) + 40;
            const x = xPosFilos[i];
            const yLabel = y(filosofo.nascimento) + padding;

            // Área de interação
            svg.append('rect')
                .attr('class', `interaction-area ${filosofo.nome.replace(/\s+/g, '-')}`)
                .attr('x', x - 25)
                .attr('y', y(filosofo.nascimento) - 20)
                .attr('width', 50)
                .attr('height', areaHeight)
                .attr('fill', 'transparent')
                .style('cursor', 'pointer')
                .on('mouseover', () => showCategoryConnections(filosofo.nome))
                .on('mouseout', () => hideCategoryConnections(filosofo.nome))
                .on('click', () => selectFilosofo(filosofo));

            // Grupo principal do rótulo
            const labelGroup = svg.append('g')
                .attr('class', `filosofo-label ${selectedFilosofo?.nome === filosofo.nome ? 'selected' : ''}`)
                .attr('transform', `translate(${x},${yLabel})`)
                .style('cursor', 'pointer')
                .on('click', () => selectFilosofo(filosofo));

            // Texto temporário para cálculo de dimensões
            const tempText = labelGroup.append('text')
                .attr('text-anchor', 'middle')
                .attr('dominant-baseline', 'middle')
                .style('font-family', 'Cinzel, serif')
                .style('font-size', `${fontSize}px`)
                .style('fill', 'transparent') // Invisível
                .text(nome);

            const bbox = tempText.node().getBBox();
            tempText.remove();

            const algumSelecionado = !!selectedFilosofo;
            const isSelected = selectedFilosofo?.nome === filosofo.nome;

            // Fatores de escala
            const shrinkFactor = 0.6;
            const widthFactor = !algumSelecionado || isSelected ? 1 : 0.65;
            const fontScale = !algumSelecionado || isSelected ? 1 : shrinkFactor;
            const paddingFactor = !algumSelecionado || isSelected ? 1 : shrinkFactor;

            // Retângulo de fundo
            labelGroup.append('rect')
                .attr('x', -bbox.width * widthFactor / 2 - padding * paddingFactor)
                .attr('y', -bbox.height / 2 - padding * paddingFactor)
                .attr('width', bbox.width * widthFactor + padding * 2 * paddingFactor)
                .attr('height', bbox.height + padding * 2 * paddingFactor)
                .attr('fill', '#fff')
                .attr('stroke', isSelected ? colors.highlight : colors.timeline)
                .attr('stroke-width', isSelected ? 2.5 : 1.2)
                .attr('rx', 6)
                .attr('ry', 6);

            // Texto visível
            labelGroup.append('text')
                .attr('text-anchor', 'middle')
                .attr('dominant-baseline', 'middle')
                .style('font-family', 'Cinzel, serif')
                .style('font-size', `${fontSize * fontScale}px`)
                .style('fill', colors.text)
                .text(nome);
        });
        
        // Marcador de morte do filósofo
        let sizeSkull = 20;
        svg.selectAll('.filosofo-end')
        .data(filosofos)
            .enter()
            .append('image')
            .attr('class', 'filosofo-end')
            .attr('x', (_, i) => xPosFilos[i] - sizeSkull / 2) 
            .attr('y', d => y(d.morte) - sizeSkull / 2)        
            .attr('width', sizeSkull)                     
            .attr('height', sizeSkull)
            .attr('href', d => 'images/skull_icon.png')
            .style('cursor', 'pointer')
            .on('click', (event, d) => selectFilosofo(d))
            .attr('class', d => `filosofo-end ${selectedFilosofo?.nome === d.nome ? 'selected' : ''}`); 
    }

    onMount(() => {
        drawTimeLine();
        window.addEventListener('resize', handleResize);

        window.addEventListener('scroll', () => {
            d3.selectAll('.category-connection')
                .attr('y1', window.scrollY + 50);
            d3.selectAll('.selected-connection')
                .attr('y1', window.scrollY + 50);
    });
    });

    afterUpdate(() => {
        drawTimeLine();

        // Arruma a posição y das conexões ao scrollar
        const scrollY = window.scrollY;
        d3.selectAll('.category-connection')
            .attr('y1', scrollY + 50);
        d3.selectAll('.selected-connection')
            .attr('y1', scrollY + 50);
    });

</script>

<div class="full-page">
    <div class="viz {isSplitView ? 'split-view' : ''}">
        <!-- Header -->
        <div class="fixed-header">
            <div class="header-content">
                <a href='{base}/' class="home-icon">
                    <!-- Home -->
                    <svg xmlns="http://www.w3.org/2000/svg" width="25" height="25" fill="currentColor" viewBox="0 0 24 24">
                        <path d="M10 20v-6h4v6h5v-8h3L12 3 2 12h3v8z"/>
                    </svg>
                </a>
                Philosophers' Timeline
            </div>
        </div>

        <!-- Categorias -->
        <div class="fixed-categories">
            <div class="interests-label">Interests:</div>
            {#each categoriaPositions as cat}
                <div class="category-label" style="left: {cat.pos}px">
                    {cat.nome}
                    <div class="tooltip">{cat.descricao}</div>
                </div>
            {/each}
        </div>

        <!-- Timeline Container -->
        <div class="container" id="timeline-container">
            <div class="fixed-year">YEAR</div>
            <svg id="timeline" class="grafico"></svg>
        </div>
    </div>

    <!-- Submódulo das Informações fos Filósofos -->
    {#if isSplitView}
        <div class="info-filosofos">
            <button on:click={closeDetailView}>Close</button>

            <h1>{selectedFilosofo.nome}</h1>

            <div class="sub-infos">
                <img src={selectedFilosofoInfo.image} alt="{selectedFilosofo.nome}">

                <div>
                    <p><strong>Lifetime: </strong>{selectedFilosofoInfo.lifetime}</p>
                    <p><strong>Branch: </strong>{selectedFilosofoInfo.branch}</p>
                    <p><strong>Fields: </strong>{selectedFilosofoInfo.fields}</p>
                </div>
            </div>

            <div class="paragraphs" on:mouseover={showTooltip} on:mouseout={hideTooltip} on:mousemove={moveTooltip}>
                {#if selectedFilosofoInfo.description.length > 0}
                    {#each selectedFilosofoInfo.description as paragraph}
                        <p>{@html processTextWithGlossary(paragraph.value, glossary)}</p>
                    {/each}
                {:else}
                    <p>No information available.</p>
                {/if}
            </div>
        </div>
    {/if}

    <!-- Glossary tooltip -->
    {#if tooltipVisible}
        <div class="glossary-tooltip" 
             style="left: {tooltipX}px; top: {tooltipY}px;">
            {tooltipContent}
        </div>
    {/if}
</div>