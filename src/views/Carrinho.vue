<template>
    <div class="tudo">
        <div class="carrinho-container">
            <!-- Header do Carrinho -->
            <div class="carrinho-header">
                <div class="titulo-secao">
                    <div class="linebranca"></div>
                    <h2>MEU CARRINHO</h2>
                    <div class="linebranca"></div>
                </div>
            </div>

            <!-- Conteúdo Principal -->
            <div class="carrinho-content">
                <!-- Carrinho Vazio -->
                <div v-if="carregando" class="estado-carrinho">
                    <div class="loading-spinner"></div>
                    <p>Carregando carrinho...</p>
                </div>

                <div v-else-if="!itensCarrinho.length" class="estado-carrinho">
                    <div class="carrinho-vazio">
                        <img src="../components/img/carrinhofinal.png" alt="Carrinho vazio" class="carrinho-vazio-img">
                        <h3>Seu carrinho está vazio</h3>
                        <p>Adicione alguns produtos para começar suas compras!</p>
                        <router-link to="/pesquisas" class="btn-continuar-comprando">
                            Continuar Comprando
                        </router-link>
                    </div>
                </div>

                <!-- Itens do Carrinho -->
                <div v-else class="carrinho-com-itens">
                    <div class="itens-container">
                        <div class="itens-lista">
                            <div v-for="item in itensCarrinho" :key="item.id" class="item-carrinho">
                                <div class="item-imagem">
                                    <img :src="item.image_path" :alt="item.name" />
                                </div>
                                <div class="item-info">
                                    <h4>{{ item.name }}</h4>
                                    <p class="item-preco">R$ {{ formatarPreco(item.unit_price) }}</p>
                                </div>
                                <div class="item-quantidade">
                                    <div class="quantidade-controles">
                                        <button 
                                            @click="diminuirQuantidade(item)" 
                                            :disabled="item.quantity <= 1"
                                            class="btn-quantidade"
                                        >
                                            -
                                        </button>
                                        <span class="quantidade-valor">{{ item.quantity }}</span>
                                        <button 
                                            @click="aumentarQuantidade(item)" 
                                            class="btn-quantidade"
                                        >
                                            +
                                        </button>
                                    </div>
                                </div>
                                <div class="item-subtotal">
                                    <p class="subtotal-valor">R$ {{ formatarPreco(item.unit_price * item.quantity) }}</p>
                                </div>
                                <div class="item-acoes">
                                    <button @click="removerItem(item)" class="btn-remover">
                                        ✕
                                    </button>
                                </div>
                            </div>
                        </div>
                    </div>

                    <!-- Resumo do Carrinho -->
                    <div class="resumo-carrinho">
                        <div class="resumo-header">
                            <h3>Resumo do Pedido</h3>
                        </div>
                        <div class="resumo-itens">
                            <div class="resumo-item">
                                <span>Subtotal ({{ totalItens }} {{ totalItens === 1 ? 'item' : 'itens' }})</span>
                                <span>R$ {{ formatarPreco(subtotal) }}</span>
                            </div>
                            <div class="resumo-item">
                                <span>Frete</span>
                                <span>Grátis</span>
                            </div>
                            <div class="resumo-total">
                                <span>Total</span>
                                <span>R$ {{ formatarPreco(total) }}</span>
                            </div>
                        </div>
                        <div class="resumo-acoes">
                            <button @click="limparCarrinho" class="btn-limpar-carrinho">
                                Limpar Carrinho
                            </button>
                            <button @click="finalizarCompra" class="btn-finalizar-compra">
                                Finalizar Compra
                            </button>
                        </div>
                        <div class="resumo-continuar">
                            <router-link to="/pesquisas" class="link-continuar-comprando">
                                ← Continuar Comprando
                            </router-link>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { useToast } from 'vue-toastification'
import api, { 
    getItensCarrinho, 
    atualizarQuantidadeCarrinho, 
    removerItemCarrinho, 
    limparCarrinho as limparCarrinhoAPI 
} from '../services/api'

const router = useRouter()
const toast = useToast()

// Estados
const carregando = ref(true)
const itensCarrinho = ref([])

// Computed
const totalItens = computed(() => {
    return itensCarrinho.value.reduce((total, item) => total + item.quantity, 0)
})

const subtotal = computed(() => {
    return itensCarrinho.value.reduce((total, item) => {
        return total + (item.unit_price * item.quantity)
    }, 0)
})

const total = computed(() => {
    return subtotal.value // Frete grátis
})

// Funções
function formatarPreco(preco) {
    return preco.toFixed(2).replace('.', ',')
}

async function carregarCarrinho() {
    try {
        carregando.value = true
        const dadosCarrinho = await getItensCarrinho()
        // Processar imagens dos itens do carrinho
        itensCarrinho.value = (dadosCarrinho.items || []).map(item => ({
            ...item,
            image_path: item.image_path && !item.image_path.startsWith('http')
                ? `http://35.196.79.227:8000${item.image_path}`
                : item.image_path
        }))
    } catch (error) {
        console.error('Erro ao carregar carrinho:', error)
        toast.error('Erro ao carregar carrinho')
        itensCarrinho.value = []
    } finally {
        carregando.value = false
    }
}

async function aumentarQuantidade(item) {
    try {
        const novaQuantidade = item.quantity + 1
        await atualizarQuantidadeCarrinho(item.product_id, novaQuantidade)
        item.quantity = novaQuantidade
        toast.success('Quantidade atualizada!')
    } catch (error) {
        console.error('Erro ao aumentar quantidade:', error)
        toast.error('Erro ao atualizar quantidade')
    }
}

async function diminuirQuantidade(item) {
    if (item.quantity <= 1) return
    try {
        const novaQuantidade = item.quantity - 1
        await atualizarQuantidadeCarrinho(item.product_id, novaQuantidade)
        item.quantity = novaQuantidade
        toast.success('Quantidade atualizada!')
    } catch (error) {
        console.error('Erro ao diminuir quantidade:', error)
        toast.error('Erro ao atualizar quantidade')
    }
}

async function removerItem(item) {
    try {
        await removerItemCarrinho(item.product_id)
        itensCarrinho.value = itensCarrinho.value.filter(i => i.id !== item.id)
        toast.success('Item removido do carrinho!')
    } catch (error) {
        console.error('Erro ao remover item:', error)
        toast.error('Erro ao remover item')
    }
}

async function limparCarrinho() {
    try {
        await limparCarrinhoAPI()
        itensCarrinho.value = []
        toast.success('Carrinho limpo com sucesso!')
    } catch (error) {
        console.error('Erro ao limpar carrinho:', error)
        toast.error('Erro ao limpar carrinho')
    }
}

function finalizarCompra() {
    if (itensCarrinho.value.length === 0) {
        toast.error('Carrinho vazio!')
        return
    }
    toast.info('Funcionalidade de checkout em desenvolvimento!')
}

onMounted(async () => {
    await carregarCarrinho()
})

window.addEventListener('carrinho-atualizado', carregarCarrinho)
</script>

<style scoped>
/* ===== LAYOUT PRINCIPAL ===== */
.tudo {
    width: 100%;
    height: 100vh;
    background-color: #ffffff;
    overflow: hidden;
    display: flex;
    justify-content: center;
    align-items: flex-start;
    padding: 20px;
    box-sizing: border-box;
}

.carrinho-container {
    width: 100%;
    max-width: 1200px;
    height: 100%;
    display: flex;
    flex-direction: column;
}

.carrinho-header {
    margin-bottom: 30px;
}

.titulo-secao {
    display: flex;
    justify-content: space-between;
    align-items: center;
    background-color: #4f79a3;
    height: 4.5rem;
    width: 100%;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
    border-radius: 8px;
}

.titulo-secao h2 {
    font-size: 26px;
    color: rgb(255, 255, 255);
    margin: 0;
    font-weight: bold;
}

.linebranca {
    width: 20vw;
    height: 1px;
    background-color: #ffffff;
}

.carrinho-content {
    flex: 1;
    display: flex;
    gap: 30px;
    overflow: hidden;
}

.estado-carrinho {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    width: 100%;
    height: 100%;
    text-align: center;
}

.loading-spinner {
    width: 50px;
    height: 50px;
    border: 4px solid #f3f3f3;
    border-top: 4px solid #4f79a3;
    border-radius: 50%;
    animation: spin 1s linear infinite;
    margin-bottom: 20px;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}

.carrinho-vazio {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 20px;
}

.carrinho-vazio-img {
    width: 120px;
    height: auto;
    opacity: 0.6;
}

.carrinho-vazio h3 {
    font-size: 24px;
    color: #333;
    margin: 0;
}

.carrinho-vazio p {
    font-size: 16px;
    color: #666;
    margin: 0;
}

.btn-continuar-comprando {
    background-color: #4f79a3;
    color: white;
    padding: 12px 24px;
    border-radius: 8px;
    text-decoration: none;
    font-weight: bold;
    transition: background-color 0.3s;
}

.btn-continuar-comprando:hover {
    background-color: #3a5a7a;
}

.carrinho-com-itens {
    display: flex;
    gap: 30px;
    width: 100%;
    height: 100%;
}

.itens-container {
    flex: 1;
    overflow-y: auto;
    padding-right: 20px;
}

.itens-lista {
    display: flex;
    flex-direction: column;
    gap: 20px;
}

.item-carrinho {
    display: flex;
    align-items: center;
    gap: 20px;
    padding: 20px;
    background-color: #f8f9fa;
    border-radius: 12px;
    border: 1px solid #e9ecef;
    transition: all 0.3s ease;
}

.item-carrinho:hover {
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
    transform: translateY(-2px);
}

.item-imagem {
    flex-shrink: 0;
}

.item-imagem img {
    width: 100px;
    height: 140px;
    object-fit: cover;
    border-radius: 8px;
    border: 1px solid #dee2e6;
}

.item-info {
    flex: 1;
    min-width: 0;
}

.item-info h4 {
    font-size: 18px;
    color: #333;
    margin: 0 0 8px 0;
    font-weight: bold;
}

.item-preco {
    font-size: 16px;
    color: #666;
    margin: 0;
}

.item-quantidade {
    flex-shrink: 0;
}

.quantidade-controles {
    display: flex;
    align-items: center;
    gap: 12px;
    background-color: white;
    border-radius: 8px;
    padding: 8px;
    border: 1px solid #dee2e6;
}

.btn-quantidade {
    width: 32px;
    height: 32px;
    border: none;
    background-color: #4f79a3;
    color: white;
    border-radius: 6px;
    cursor: pointer;
    font-size: 18px;
    font-weight: bold;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: background-color 0.3s;
}

.btn-quantidade:hover:not(:disabled) {
    background-color: #3a5a7a;
}

.btn-quantidade:disabled {
    background-color: #ccc;
    cursor: not-allowed;
}

.quantidade-valor {
    font-size: 16px;
    font-weight: bold;
    color: #333;
    min-width: 30px;
    text-align: center;
}

.item-subtotal {
    flex-shrink: 0;
    text-align: right;
}

.subtotal-valor {
    font-size: 18px;
    font-weight: bold;
    color: #333;
    margin: 0;
}

.item-acoes {
    flex-shrink: 0;
}

.btn-remover {
    width: 40px;
    height: 40px;
    border: none;
    background-color: #dc3545;
    color: white;
    border-radius: 8px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: background-color 0.3s;
    font-size: 18px;
    font-weight: bold;
}

.btn-remover:hover {
    background-color: #c82333;
}

.resumo-carrinho {
    width: 350px;
    background-color: #f8f9fa;
    border-radius: 12px;
    border: 1px solid #e9ecef;
    padding: 24px;
    height: fit-content;
    position: sticky;
    top: 20px;
}

.resumo-header h3 {
    font-size: 20px;
    color: #333;
    margin: 0 0 20px 0;
    font-weight: bold;
    text-align: center;
}

.resumo-itens {
    margin-bottom: 24px;
}

.resumo-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 12px 0;
    border-bottom: 1px solid #e9ecef;
    font-size: 16px;
    color: #666;
}

.resumo-item:last-of-type {
    border-bottom: none;
}

.resumo-total {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 16px 0;
    border-top: 2px solid #dee2e6;
    margin-top: 16px;
    font-size: 18px;
    font-weight: bold;
    color: #333;
}

.resumo-acoes {
    display: flex;
    flex-direction: column;
    gap: 12px;
    margin-bottom: 20px;
}

.btn-limpar-carrinho {
    padding: 12px 20px;
    background-color: #6c757d;
    color: white;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    font-size: 16px;
    font-weight: bold;
    transition: background-color 0.3s;
}

.btn-limpar-carrinho:hover {
    background-color: #5a6268;
}

.btn-finalizar-compra {
    padding: 16px 20px;
    background-color: #28a745;
    color: white;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    font-size: 18px;
    font-weight: bold;
    transition: background-color 0.3s;
}

.btn-finalizar-compra:hover {
    background-color: #218838;
}

.resumo-continuar {
    text-align: center;
}

.link-continuar-comprando {
    color: #4f79a3;
    text-decoration: none;
    font-size: 16px;
    font-weight: bold;
    transition: color 0.3s;
}

.link-continuar-comprando:hover {
    color: #3a5a7a;
    text-decoration: underline;
}

@media (max-width: 1024px) {
    .carrinho-com-itens {
        flex-direction: column;
    }
    .resumo-carrinho {
        width: 100%;
        position: static;
    }
    .itens-container {
        padding-right: 0;
    }
}

@media (max-width: 768px) {
    .tudo {
        padding: 10px;
    }
    .titulo-secao h2 {
        font-size: 20px;
    }
    .linebranca {
        width: 15vw;
    }
    .item-carrinho {
        flex-direction: column;
        text-align: center;
        gap: 15px;
    }
    .item-imagem img {
        width: 80px;
        height: 112px;
    }
    .quantidade-controles {
        justify-content: center;
    }
    .item-subtotal {
        text-align: center;
    }
    .resumo-carrinho {
        padding: 20px;
    }
}

@media (max-width: 480px) {
    .tudo {
        padding: 5px;
    }
    .titulo-secao {
        height: 3.5rem;
    }
    .titulo-secao h2 {
        font-size: 16px;
    }
    .linebranca {
        width: 10vw;
    }
    .carrinho-content {
        gap: 20px;
    }
    .item-carrinho {
        padding: 15px;
    }
    .item-imagem img {
        width: 60px;
        height: 84px;
    }
    .item-info h4 {
        font-size: 16px;
    }
    .item-preco {
        font-size: 14px;
    }
    .subtotal-valor {
        font-size: 16px;
    }
    .quantidade-controles {
        gap: 8px;
    }
    .btn-quantidade {
        width: 28px;
        height: 28px;
        font-size: 16px;
    }
    .quantidade-valor {
        font-size: 14px;
    }
    .btn-remover {
        width: 36px;
        height: 36px;
    }
    .resumo-carrinho {
        padding: 15px;
    }
    .resumo-header h3 {
        font-size: 18px;
    }
    .resumo-item {
        font-size: 14px;
    }
    .resumo-total {
        font-size: 16px;
    }
    .btn-limpar-carrinho,
    .btn-finalizar-compra {
        font-size: 14px;
        padding: 12px 16px;
    }
    .link-continuar-comprando {
        font-size: 14px;
    }
}

@media (max-width: 360px) {
    .item-carrinho {
        padding: 10px;
    }
    .item-imagem img {
        width: 50px;
        height: 70px;
    }
    .item-info h4 {
        font-size: 14px;
    }
    .item-preco {
        font-size: 12px;
    }
    .btn-quantidade {
        width: 24px;
        height: 24px;
        font-size: 14px;
    }
    .quantidade-valor {
        font-size: 12px;
    }
    .subtotal-valor {
        font-size: 14px;
    }
    .btn-remover {
        width: 32px;
        height: 32px;
    }
}
</style>