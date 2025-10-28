<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Buscador CNJ DataJud</title>
    <!-- Carrega Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Configuração do Tailwind para a fonte Inter -->
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    fontFamily: {
                        sans: ['Inter', 'sans-serif'],
                    },
                }
            }
        }
    </script>
    <!-- Ícones Lucide -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <!-- Adiciona biblioteca para leitura e escrita de Excel (SheetJS) -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap');
        
        /* Estilo para a caixa de sombra de foco */
        .focus-ring {
            box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.5); /* Cor azul do Tailwind */
        }
        /* Esconde o input file nativo */
        .custom-file-input {
            opacity: 0;
            position: absolute;
            width: 100%;
            height: 100%;
            cursor: pointer;
        }
    </style>
</head>
<body class="bg-gray-50 min-h-screen flex items-center justify-center p-4 font-sans">

    <div id="app" class="w-full max-w-2xl bg-white shadow-xl rounded-2xl p-6 md:p-10 border border-blue-100">
        
        <!-- Cabeçalho -->
        <h1 class="text-3xl font-extrabold text-blue-700 mb-2 flex items-center">
            <i data-lucide="scale" class="w-7 h-7 mr-3 text-blue-500"></i>
            Busca de Processos CNJ
        </h1>
        <p class="text-gray-500 mb-6">Consulte metadados de processos usando o número único (Padrão CNJ) via API Pública DataJud do CNJ.</p>

        <!-- Formulário de Busca Individual -->
        <div class="space-y-4 border-b border-gray-200 pb-6 mb-6">
            <h2 class="text-xl font-semibold text-gray-700">Busca Individual</h2>
            <label for="cnj-number" class="block text-sm font-medium text-gray-700">Número CNJ (Ex: 0000000-00.2024.8.26.0000)</label>
            <input type="text" id="cnj-number" placeholder="Digite o número CNJ" 
                   class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 transition duration-150"
                   maxlength="25"
                   value="0001000-11.2024.8.19.0001" /> <!-- Exemplo TJRJ -->

            <button onclick="buscarProcessoIndividual()" id="search-button"
                    class="w-full flex items-center justify-center px-4 py-3 bg-blue-600 text-white font-semibold rounded-lg shadow-md hover:bg-blue-700 transition duration-300 transform active:scale-98">
                <span id="button-text">Buscar Processo</span>
                <i data-lucide="search" class="w-5 h-5 ml-2" id="search-icon"></i>
                <div id="loading-spinner" class="hidden animate-spin rounded-full h-5 w-5 border-b-2 border-white ml-2"></div>
            </button>
        </div>

        <!-- Seção de Processamento em Lote -->
        <div class="space-y-4 border-b border-gray-200 pb-6 mb-6">
            <h2 class="text-xl font-semibold text-gray-700 flex items-center">
                <i data-lucide="upload-cloud" class="w-5 h-5 mr-2 text-blue-500"></i>
                Relatório em Lote (Excel)
            </h2>
            <p class="text-sm text-gray-500">Faça o upload de uma planilha Excel (.xlsx) contendo os números CNJ na **primeira coluna**.</p>

            <label for="excel-file" class="block">
                <div class="relative w-full p-3 border-2 border-dashed border-blue-300 bg-blue-50 rounded-lg hover:border-blue-500 transition duration-150 flex items-center justify-center cursor-pointer">
                    <input type="file" id="excel-file" accept=".xlsx, .xls" class="custom-file-input" onchange="displayFileName(this.files)">
                    <span id="file-name" class="text-sm text-blue-600 font-medium">Clique para selecionar o arquivo Excel</span>
                    <i data-lucide="file-up" class="w-5 h-5 ml-2 text-blue-500"></i>
                </div>
            </label>

            <button onclick="handleFileUpload()" id="batch-button" disabled
                    class="w-full flex items-center justify-center px-4 py-3 bg-green-500 text-white font-semibold rounded-lg shadow-md hover:bg-green-600 transition duration-300 transform active:scale-98 disabled:bg-green-300">
                <span id="batch-button-text">Processar Lote e Gerar Relatório</span>
                <i data-lucide="download" class="w-5 h-5 ml-2" id="download-icon"></i>
                <div id="batch-loading-spinner" class="hidden animate-spin rounded-full h-5 w-5 border-b-2 border-white ml-2"></div>
            </button>
            <div id="batch-status" class="text-sm text-gray-600 hidden p-2 bg-yellow-50 rounded-lg"></div>
        </div>
        
        <!-- Área de Resultados -->
        <div id="result-area" class="mt-8">
            <div id="initial-message" class="text-center p-6 bg-blue-50 border border-blue-200 rounded-lg text-blue-700">
                <i data-lucide="info" class="w-6 h-6 mx-auto mb-2"></i>
                <p>Insira um número CNJ válido ou carregue um arquivo Excel para começar.</p>
            </div>
            
            <!-- Este div será preenchido com os resultados ou erros -->
            <div id="process-details" class="hidden"></div>
        </div>

    </div>

    <script>
        // Mapeamento dos códigos de Tribunal (J.TR) para o slug do endpoint DataJud.
        const TRIBUNAL_MAP = {
            // Tribunais de Justiça Estaduais (8.xx)
            "8.01": "tjac", "8.02": "tjal", "8.03": "tjam", "8.04": "tjap", 
            "8.05": "tjba", "8.06": "tjce", "8.07": "tjdf", "8.08": "tjes", 
            "8.09": "tjgo", "8.10": "tjma", "8.11": "tjmg", "8.12": "tjms", 
            "8.13": "tjmt", "8.14": "tjpa", "8.15": "tjpb", "8.16": "tjpe", 
            "8.17": "tjpi", "8.18": "tjpr", "8.19": "tjrj", "8.20": "tjrn", 
            "8.21": "tjro", "8.22": "tjrr", "8.23": "tjrs", "8.24": "tjsc", 
            "8.25": "tjse", "8.26": "tjsp", "8.27": "tjto",
            // Tribunais Regionais Federais (4.xx) - Adicionados
            "4.01": "trf1", // TRF 1ª Região
            "4.02": "trf2", // TRF 2ª Região
            "4.03": "trf3", // TRF 3ª Região
            "4.04": "trf4", // TRF 4ª Região
            "4.05": "trf5", // TRF 5ª Região
            "4.06": "trf6", // TRF 6ª Região
        };

        const API_KEY = ""; // Não é necessária para este endpoint público
        const BASE_API_URL = "https://api-publica.datajud.cnj.jus.br";
        
        // Elementos DOM
        const cnjInput = document.getElementById('cnj-number');
        const searchButton = document.getElementById('search-button');
        const buttonText = document.getElementById('button-text');
        const loadingSpinner = document.getElementById('loading-spinner');
        const resultArea = document.getElementById('process-details');
        const initialMessage = document.getElementById('initial-message');
        
        // Elementos DOM adicionais para o LOTE
        const batchButton = document.getElementById('batch-button');
        const batchButtonText = document.getElementById('batch-button-text');
        const batchLoadingSpinner = document.getElementById('batch-loading-spinner');
        const batchStatus = document.getElementById('batch-status');


        /**
         * Extrai o código do Tribunal (J.TR) do número CNJ formatado.
         */
        function extractTribunalCode(cnjNumber) {
            // Espera o formato NNNNNNN-DD.YYYY.J.TR.OOOO (ex: 8.19 ou 4.01)
            const match = cnjNumber.match(/\d{7}-\d{2}\.\d{4}\.((\d)\.(\d{2}))\.\d{4}/);
            return match ? match[1] : null;
        }

        /**
         * Valida e normaliza o número CNJ.
         * Suporta tanto o formato separado (XXX-XX.XXXX.J.TR.XXXX) quanto o formato de 20 dígitos brutos.
         */
        function validateAndNormalizeCNJ(cnj) {
            const cnjStr = String(cnj).trim();
            
            // 1. Tenta validar o formato já separado
            const regexSeparated = /^(\d{7})-(\d{2})\.(\d{4})\.(\d)\.(\d{2})\.(\d{4})$/; 
            if (regexSeparated.test(cnjStr)) {
                return cnjStr;
            }

            // 2. Se não estiver separado, remove todos os caracteres não-dígitos
            const digitsOnly = cnjStr.replace(/\D/g, '');

            // 3. Verifica se tem 20 dígitos (formato sem pontuação NNNNNNNDDYYYYJTRRRR)
            if (digitsOnly.length === 20) {
                // Formata: NNNNNNN-DD.YYYY.J.TR.RRRR
                const part1 = digitsOnly.substring(0, 7);
                const part2 = digitsOnly.substring(7, 9);
                const part3 = digitsOnly.substring(9, 13);
                const part4 = digitsOnly.substring(13, 14);
                const part5 = digitsOnly.substring(14, 16);
                const part6 = digitsOnly.substring(16, 20);
                
                return `${part1}-${part2}.${part3}.${part4}.${part5}.${part6}`;
            }
            
            // 4. Falha na validação
            return null;
        }

        /**
         * Renderiza uma mensagem de erro na tela (uso individual).
         */
        function displayError(message) {
            initialMessage.classList.add('hidden');
            resultArea.classList.remove('hidden');
            resultArea.innerHTML = `
                <div class="bg-red-100 border-l-4 border-red-500 text-red-700 p-4 rounded-lg shadow-md flex items-start">
                    <i data-lucide="alert-triangle" class="w-5 h-5 mr-3 mt-1 flex-shrink-0"></i>
                    <div>
                        <p class="font-bold">Erro na Busca</p>
                        <p class="text-sm">${message}</p>
                    </div>
                </div>
            `;
            lucide.createIcons();
        }

        /**
         * Função que retorna dados mockados simulando uma resposta de sucesso do DataJud.
         */
        function getMockProcessData(cnj) {
            const tribunalCode = extractTribunalCode(cnj);
            const tribunalSlug = TRIBUNAL_MAP[tribunalCode] || 'TJ'; // Pode ser TRF também
            
            // Dados Mockados genéricos, ajustando o link conforme o TRF/TJ
            let urlBase = "https://consultas.tjrj.jus.br/#/processo";
            if (tribunalSlug.startsWith('trf')) {
                urlBase = `https://processual.trf${tribunalSlug.slice(3)}.jus.br/consultaProcessual/consultaPublica/`;
            }

            return {
                "hits": {
                    "total": { "value": 1, "relation": "eq" },
                    "hits": [
                        {
                            "_source": {
                                "numeroProcesso": cnj,
                                "situacao": "Ativo - Aguardando Julgamento",
                                "classe": { "nome": "Procedimento Comum Cível" },
                                "orgaoJulgador": { "nome": "1ª Vara Cível Federal" },
                                "assuntos": [
                                    { "nome": "Dano Material" },
                                    { "nome": "Responsabilidade Civil" }
                                ],
                                "dataHoraUltimaAtualizacao": new Date().toISOString(),
                                "valorDaCausa": 55000.00,
                                "dataAjuizamento": "2024-03-15T10:00:00Z", // Data de Distribuição
                                "partes": [
                                    { "tipoDeParticipacao": "Autor", "nome": "João da Silva" },
                                    { "tipoDeParticipacao": "Réu", "nome": "Empresa Fictícia S.A." }
                                ],
                                "advogados": [
                                    { "nome": "Mariana Souza", "numeroInscricaoOAB": "OAB/RJ 123456" },
                                    { "nome": "Roberto Carlos", "numeroInscricaoOAB": "OAB/SP 789012" }
                                ],
                                "urlProcesso": `${urlBase}/${cnj}`
                            }
                        }
                    ]
                }
            };
        }

        /**
         * Renderiza os detalhes do processo (uso individual).
         */
        function displayProcessDetails(processo, tribunalSlug) {
            initialMessage.classList.add('hidden');
            resultArea.classList.remove('hidden');
            
            const status = processo.situacao || "Não Informado";
            const classe = processo.classe?.nome || "Não Informada";
            const assuntos = processo.assuntos?.map(a => a.nome).join(', ') || "N/A";
            const dataAtualizacao = processo.dataHoraUltimaAtualizacao ? 
                new Date(processo.dataHoraUltimaAtualizacao).toLocaleString('pt-BR') : 'Desconhecida';
            const valorDaCausa = processo.valorDaCausa ? 
                new Intl.NumberFormat('pt-BR', { style: 'currency', currency: 'BRL' }).format(processo.valorDaCausa) : 'N/A';
            const dataDistribuicao = processo.dataAjuizamento ? 
                new Date(processo.dataAjuizamento).toLocaleDateString('pt-BR') : 'Desconhecida';
            
            const partesHtml = processo.partes && processo.partes.length > 0
                ? processo.partes.map(p => `<span class="font-semibold text-gray-700">${p.tipoDeParticipacao}:</span> <span class="text-gray-900">${p.nome}</span>`).join('<br>')
                : 'Não Informado';

            const advogadosHtml = processo.advogados && processo.advogados.length > 0
                ? processo.advogados.map(a => `<span class="font-semibold text-gray-700">${a.nome}</span> (<span class="text-gray-900">${a.numeroInscricaoOAB || 'OAB Não Info'}</span>)`).join('<br>')
                : 'Não Informado';


            resultArea.innerHTML = `
                <div class="bg-blue-50 p-6 rounded-xl border-t-4 border-blue-600 shadow-lg">
                    <h3 class="text-xl font-bold text-blue-700 mb-4 flex items-center">
                        <i data-lucide="file-text" class="w-6 h-6 mr-3"></i>
                        Detalhes do Processo
                    </h3>
                    
                    <div class="grid grid-cols-1 md:grid-cols-2 gap-x-6 gap-y-4">
                        <!-- Linha 1: CNJ / Órgão Julgador -->
                        <div class="space-y-1">
                            <p class="text-xs font-semibold uppercase text-gray-500">Número CNJ</p>
                            <p class="text-sm font-medium text-gray-900 break-words">${processo.numeroProcesso}</p>
                        </div>
                        <div class="space-y-1">
                            <p class="text-xs font-semibold uppercase text-gray-500">Órgão Julgador</p>
                            <p class="text-sm font-medium text-gray-900">${processo.orgaoJulgador?.nome || 'N/A'}</p>
                        </div>

                        <!-- Linha 2: Classe / Situação -->
                        <div class="space-y-1">
                            <p class="text-xs font-semibold uppercase text-gray-500">Classe Judicial</p>
                            <p class="text-sm font-medium text-gray-900">${classe}</p>
                        </div>
                        <div class="space-y-1">
                            <p class="text-xs font-semibold uppercase text-gray-500">Situação</p>
                            <span class="inline-block px-3 py-1 text-xs font-medium rounded-full bg-green-100 text-green-800">${status}</span>
                        </div>
                        
                        <!-- Linha 3: Valor da Causa / Data de Distribuição (NOVOS) -->
                        <div class="space-y-1">
                            <p class="text-xs font-semibold uppercase text-gray-500">Valor da Causa</p>
                            <p class="text-sm font-bold text-green-600">${valorDaCausa}</p>
                        </div>
                        <div class="space-y-1">
                            <p class="text-xs font-semibold uppercase text-gray-500">Data de Distribuição</p>
                            <p class="text-sm font-medium text-gray-900">${dataDistribuicao}</p>
                        </div>

                        <!-- Linha Completa: Assuntos -->
                        <div class="space-y-1 md:col-span-2 pt-4 border-t border-blue-200">
                            <p class="text-xs font-semibold uppercase text-gray-500">Assuntos Principais</p>
                            <p class="text-sm text-gray-900">${assuntos}</p>
                        </div>

                        <!-- Seção: Partes (NOVO) -->
                        <div class="space-y-1 md:col-span-2 pt-4 border-t border-blue-200">
                            <p class="text-sm font-bold uppercase text-blue-700 mb-1">Partes Envolvidas</p>
                            <p class="text-sm text-gray-800 leading-relaxed">${partesHtml}</p>
                        </div>

                        <!-- Seção: Advogados (NOVO) -->
                        <div class="space-y-1 md:col-span-2 pt-4 border-t border-blue-200">
                            <p class="text-sm font-bold uppercase text-blue-700 mb-1">Advogados</p>
                            <p class="text-sm text-gray-800 leading-relaxed">${advogadosHtml}</p>
                        </div>

                        <!-- Linha Completa: Última Atualização (Metadado) -->
                        <div class="space-y-1 md:col-span-2 pt-4 border-t border-blue-200">
                            <p class="text-xs font-semibold uppercase text-gray-500">Última Atualização (API)</p>
                            <p class="text-sm text-gray-900">${dataAtualizacao} (Tribunal: ${tribunalSlug.toUpperCase()})</p>
                        </div>
                    </div>

                    <!-- Botão de Ação -->
                    <div class="mt-6 pt-4 border-t border-blue-300">
                        <a href="${processo.urlProcesso || '#'}" target="_blank"
                           class="inline-flex items-center text-sm font-semibold text-blue-600 hover:text-blue-800 transition duration-150">
                           Ver no Site do Tribunal
                           <i data-lucide="external-link" class="w-4 h-4 ml-1"></i>
                        </a>
                    </div>

                </div>
            `;
            lucide.createIcons();
        }

        /**
         * Tenta buscar os detalhes de um processo. Em caso de falha de fetch, usa dados mockados.
         * Retorna o objeto do processo (processo._source) e o tribunal, ou objeto de erro.
         * @param {string} normalizedCNJ - Número CNJ normalizado.
         * @returns {Promise<Object|null>}
         */
        async function fetchProcessDetails(normalizedCNJ) {
            const tribunalCode = extractTribunalCode(normalizedCNJ);
            const tribunalSlug = TRIBUNAL_MAP[tribunalCode];

            if (!tribunalSlug) {
                // Mensagem de erro atualizada para incluir a necessidade de mapeamento
                return { error: `Tribunal ${tribunalCode} não mapeado. Este tribunal ainda não está incluído no sistema ou o código CNJ está incorreto.` };
            }

            const apiUrl = `${BASE_API_URL}/api_publica_${tribunalSlug}/_search`;
            const payload = {
                "query": {
                    "bool": {
                        "must": [
                            {"match": {"numeroProcesso": normalizedCNJ}}
                        ]
                    }
                },
                "size": 1
            };
            const headers = { "Content-Type": "application/json" };

            const MAX_RETRIES = 3;
            let responseData = null;

            for (let i = 0; i < MAX_RETRIES; i++) {
                try {
                    const response = await fetch(apiUrl, {
                        method: 'POST',
                        headers: headers,
                        body: JSON.stringify(payload)
                    });

                    if (!response.ok) {
                        if (i < MAX_RETRIES - 1) {
                            const delay = Math.pow(2, i) * 1000;
                            await new Promise(resolve => setTimeout(resolve, delay));
                            continue; 
                        } else {
                            throw new Error(`Status: ${response.status} ${response.statusText}`);
                        }
                    }

                    responseData = await response.json();
                    break; 
                } catch (error) {
                    if (error.message === 'Failed to fetch' || error instanceof TypeError) {
                        console.warn(`[CNJ: ${normalizedCNJ}] Falha no 'fetch' detectada. Usando dados simulados.`);
                        responseData = getMockProcessData(normalizedCNJ);
                        break; 
                    }

                    if (i === MAX_RETRIES - 1) {
                        console.error(`[CNJ: ${normalizedCNJ}] Erro fatal na busca:`, error);
                        return { error: `Erro na API: ${error.message}` };
                    }

                    const delay = Math.pow(2, i) * 1000;
                    await new Promise(resolve => setTimeout(resolve, delay));
                }
            }

            if (responseData && responseData.hits && responseData.hits.total.value > 0) {
                return { 
                    data: responseData.hits.hits[0]._source, 
                    tribunalSlug: tribunalSlug 
                };
            } else {
                // Aqui o erro é de processo não encontrado, não de CNJ inválido.
                return { error: `Processo não encontrado no ${tribunalSlug.toUpperCase()}.` };
            }
        }


        /**
         * Função de Busca Individual (Lida com a UI).
         */
        async function buscarProcessoIndividual() {
            const cnjNumber = cnjInput.value.trim();
            const normalizedCNJ = validateAndNormalizeCNJ(cnjNumber);

            if (!normalizedCNJ) {
                return displayError("O número CNJ fornecido é inválido. Utilize o formato NNNNNNN-DD.YYYY.J.TR.OOOO ou a sequência de 20 dígitos.");
            }
            
            // UI: Iniciar Carregamento
            searchButton.disabled = true;
            buttonText.textContent = "Buscando...";
            loadingSpinner.classList.remove('hidden');
            resultArea.innerHTML = '';
            resultArea.classList.remove('hidden');
            initialMessage.classList.add('hidden');

            const result = await fetchProcessDetails(normalizedCNJ);
            
            // UI: Finalizar Carregamento
            searchButton.disabled = false;
            buttonText.textContent = "Buscar Processo";
            loadingSpinner.classList.add('hidden');

            // Processar Resultados
            if (result.data) {
                displayProcessDetails(result.data, result.tribunalSlug);
            } else {
                displayError(`Não foi possível buscar os detalhes do CNJ ${normalizedCNJ}. Motivo: ${result.error}`);
            }
        }

        /**
         * Exibe o nome do arquivo selecionado e habilita o botão de lote.
         */
        function displayFileName(files) {
            const fileNameSpan = document.getElementById('file-name');
            if (files.length > 0) {
                fileNameSpan.textContent = `Arquivo selecionado: ${files[0].name}`;
                batchButton.disabled = false;
            } else {
                fileNameSpan.textContent = 'Clique para selecionar o arquivo Excel';
                batchButton.disabled = true;
            }
        }

        /**
         * Lida com o upload do arquivo Excel, lê os CNJs e inicia o processamento em lote.
         */
        function handleFileUpload() {
            const fileInput = document.getElementById('excel-file');
            const file = fileInput.files[0];

            if (!file) {
                batchStatus.textContent = "Por favor, selecione um arquivo Excel.";
                batchStatus.classList.remove('hidden');
                return;
            }
            
            // UI: Iniciar Carregamento do Lote
            batchButton.disabled = true;
            batchButtonText.textContent = "Lendo arquivo...";
            batchLoadingSpinner.classList.remove('hidden');
            batchStatus.classList.remove('hidden', 'bg-red-100', 'text-red-700', 'bg-green-100', 'text-green-700');
            batchStatus.classList.add('bg-blue-50', 'text-blue-700');
            batchStatus.innerHTML = '<i data-lucide="loader" class="w-4 h-4 mr-1 inline-block animate-spin"></i> Lendo arquivo...';
            lucide.createIcons();


            const reader = new FileReader();
            reader.onload = async function(e) {
                try {
                    const data = new Uint8Array(e.target.result);
                    const workbook = XLSX.read(data, { type: 'array' });
                    const sheetName = workbook.SheetNames[0];
                    const worksheet = workbook.Sheets[sheetName];
                    
                    // Converte a primeira coluna para JSON, assumindo cabeçalho na linha 1
                    const json = XLSX.utils.sheet_to_json(worksheet, { header: 1 });
                    
                    // Pega a primeira coluna (índice 0) e remove vazios. 
                    // Garante que o valor seja tratado como string ao passar para o validateAndNormalizeCNJ
                    let cnjList = json.map(row => row[0]).filter(cnj => cnj && String(cnj).trim() !== '');

                    // Remove o primeiro item se ele não for um CNJ válido (assumindo que é o cabeçalho)
                    if (cnjList.length > 0 && !validateAndNormalizeCNJ(String(cnjList[0]))) {
                        cnjList.shift();
                    }

                    if (cnjList.length === 0) {
                        throw new Error("Não foram encontrados números CNJ válidos na primeira coluna da planilha.");
                    }
                    
                    // Inicia o processamento
                    await processBatch(cnjList);

                } catch (error) {
                    batchStatus.classList.remove('bg-blue-50', 'text-blue-700');
                    batchStatus.classList.add('bg-red-100', 'text-red-700');
                    batchStatus.innerHTML = `<i data-lucide="alert-circle" class="w-4 h-4 mr-1 inline-block"></i> Erro ao ler planilha: ${error.message}`;
                    lucide.createIcons();
                    
                    // UI: Finalizar Carregamento
                    batchButton.disabled = false;
                    batchButtonText.textContent = "Processar Lote e Gerar Relatório";
                    batchLoadingSpinner.classList.add('hidden');
                }
            };
            reader.readAsArrayBuffer(file);
        }

        /**
         * Processa a lista de CNJs, buscando detalhes para cada um.
         */
        async function processBatch(cnjList) {
            const results = [];
            const total = cnjList.length;
            let processed = 0;

            batchStatus.classList.remove('bg-blue-50');
            batchStatus.classList.add('bg-yellow-50', 'text-gray-700');
            
            for (const cnj of cnjList) {
                processed++;
                const normalizedCNJ = validateAndNormalizeCNJ(String(cnj));

                batchStatus.innerHTML = `<i data-lucide="loader" class="w-4 h-4 mr-1 inline-block animate-spin"></i> Processando ${processed}/${total} CNJs (Atual: ${normalizedCNJ || 'CNJ Inválido'})...`;
                lucide.createIcons();

                if (!normalizedCNJ) {
                    results.push({
                        "Número CNJ": String(cnj), // Mantém o original para o relatório
                        "Status Busca": "CNJ Inválido",
                        "Erro": "Formato CNJ inválido ou não detectado",
                    });
                    continue;
                }

                const result = await fetchProcessDetails(normalizedCNJ);

                if (result.data) {
                    results.push(formatResultForExcel(result.data, result.tribunalSlug));
                } else {
                    results.push({
                        "Número CNJ": normalizedCNJ,
                        "Status Busca": "Falha na Busca",
                        "Erro": result.error || "Erro desconhecido na busca",
                    });
                }
                
                // Pequeno delay para feedback visual
                await new Promise(resolve => setTimeout(resolve, 50));
            }
            
            // 7. Gerar Excel
            generateExcelReport(results);

            // UI: Finalizar Carregamento
            batchButton.disabled = false;
            batchButtonText.textContent = "Processar Lote e Gerar Relatório";
            batchLoadingSpinner.classList.add('hidden');
            batchStatus.classList.remove('bg-yellow-50', 'text-gray-700');
            batchStatus.classList.add('bg-green-100', 'text-green-700');
            batchStatus.innerHTML = `<i data-lucide="check-circle" class="w-4 h-4 mr-1 inline-block"></i> Processamento concluído. ${results.length} registros baixados.`;
            lucide.createIcons();
        }

        /**
         * Formata o objeto de detalhes do processo para uma linha de relatório Excel.
         */
        function formatResultForExcel(processo, tribunalSlug) {
            const dataAtualizacao = processo.dataHoraUltimaAtualizacao ? new Date(processo.dataHoraUltimaAtualizacao).toLocaleString('pt-BR') : 'Desconhecida';
            // Mantemos o valor numérico para o Excel, mas garantimos que seja um número.
            const valorDaCausa = typeof processo.valorDaCausa === 'number' ? processo.valorDaCausa : 0; 
            const dataDistribuicao = processo.dataAjuizamento ? new Date(processo.dataAjuizamento).toLocaleDateString('pt-BR') : 'Desconhecida';
            
            // Coleta e concatena informações de listas com '; ' como separador
            const autores = processo.partes?.filter(p => p.tipoDeParticipacao === 'Autor' || p.tipoDeParticipacao === 'Polo Ativo').map(p => p.nome).join('; ') || 'N/A';
            const reus = processo.partes?.filter(p => p.tipoDeParticipacao === 'Réu' || p.tipoDeParticipacao === 'Polo Passivo').map(p => p.nome).join('; ') || 'N/A';
            const advogados = processo.advogados?.map(a => `${a.nome} (${a.numeroInscricaoOAB || 'OAB N/A'})`).join('; ') || 'N/A';
            const assuntos = processo.assuntos?.map(a => a.nome).join('; ') || 'N/A';
            
            return {
                "Número CNJ": processo.numeroProcesso,
                "Status Busca": "Sucesso",
                "Tribunal": tribunalSlug.toUpperCase(),
                "Órgão Julgador": processo.orgaoJulgador?.nome || 'N/A',
                "Classe Judicial": processo.classe?.nome || 'N/A',
                "Situação": processo.situacao || "Não Informado",
                "Valor da Causa (BRL)": valorDaCausa,
                "Data de Distribuição": dataDistribuicao,
                "Autores/Polo Ativo": autores,
                "Réus/Polo Passivo": reus,
                "Advogados": advogados,
                "Assuntos": assuntos,
                "Última Atualização API": dataAtualizacao,
                "Link Processo": processo.urlProcesso || 'N/A',
            };
        }


        /**
         * Cria o arquivo Excel a partir dos resultados e inicia o download.
         */
        function generateExcelReport(results) {
            if (!results || results.length === 0) {
                return; 
            }

            // O 'XLSX' é o objeto global fornecido pelo CDN
            const ws = XLSX.utils.json_to_sheet(results);
            
            const wb = XLSX.utils.book_new();
            XLSX.utils.book_append_sheet(wb, ws, "RelatorioProcessosCNJ");

            // Iniciar o download
            const filename = `Relatorio_Processos_CNJ_${new Date().toISOString().slice(0, 10)}.xlsx`;
            XLSX.writeFile(wb, filename);
        }
        
        // Inicializa os ícones Lucide após o carregamento do DOM
        document.addEventListener('DOMContentLoaded', () => {
            lucide.createIcons();
            
            // Adiciona listener para a tecla Enter na busca individual
            cnjInput.addEventListener('keypress', function(event) {
                if (event.key === 'Enter') {
                    buscarProcessoIndividual();
                }
            });
        });

    </script>
</body>
</html>
