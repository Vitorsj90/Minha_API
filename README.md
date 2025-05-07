// src/app.js

// Importa o framework Express para criar a aplicação web
const express = require('express');
// Importa o middleware de logging de requisições
const morgan = require('morgan');
// Importa o módulo de rotas da aplicação
const tarefasRoutes = require('./routes/tarefasRoutes');

// Inicializa o aplicativo Express
const app = express();

// Middleware para permitir que o Express interprete JSON
app.use(express.json());

// Middleware para logar as requisições (apenas fora de produção)
if (process.env.NODE_ENV !== 'production') app.use(morgan('dev'));

// Define o uso das rotas de tarefas na aplicação
app.use(tarefasRoutes);

// Middleware para tratar rotas inexistentes
app.use((req, res) => res.status(404).json({ erro: 'Rota não encontrada.' }));

// Inicia o servidor na porta 3000
app.listen(3000, () => {
  console.log('API rodando na porta 3000 🚀');
});

// src/routes/tarefasRoutes.js

const express = require('express');
const controller = require('../controllers/tarefasController');
const validateTarefa = require('../middlewares/validateTarefa');

const router = express.Router();

// Define as rotas da API RESTful de tarefas
router.get('/tarefas', controller.listar); // Lista todas as tarefas ou filtradas
router.get('/tarefas/:id', controller.buscar); // Busca uma tarefa por ID
router.post('/tarefas', validateTarefa, controller.criar); // Cria uma nova tarefa
router.put('/tarefas/:id', validateTarefa, controller.atualizar); // Atualiza uma tarefa
router.patch('/tarefas/:id/concluir', controller.concluir); // Marca tarefa como concluída
router.delete('/tarefas/:id', controller.deletar); // Deleta uma tarefa

module.exports = router;

// src/controllers/tarefasController.js

const service = require('../services/tarefasService');
const { log } = require('../utils/logger');

module.exports = {
  // Cria uma nova tarefa
  criar(req, res) {
    try {
      const tarefa = service.adicionarTarefa(req.body);
      log('Tarefa criada com sucesso.');
      return res.status(201).json(tarefa);
    } catch (e) {
      return res.status(500).json({ erro: 'Erro ao criar tarefa.' });
    }
  },

  // Lista todas as tarefas, com ou sem filtro por concluída
  listar(req, res) {
    const { concluida } = req.query;
    const tarefas = service.listarTarefas(concluida);
    return res.json(tarefas);
  },

  // Busca uma tarefa específica por ID
  buscar(req, res) {
    const tarefa = service.buscarPorId(req.params.id);
    if (!tarefa) return res.status(404).json({ erro: 'Tarefa não encontrada.' });
    return res.json(tarefa);
  },

  // Atualiza uma tarefa por ID
  atualizar(req, res) {
    const tarefa = service.atualizarTarefa(req.params.id, req.body);
    if (!tarefa) return res.status(404).json({ erro: 'Tarefa não encontrada.' });
    log('Tarefa atualizada.');
    return res.json(tarefa);
  },

  // Deleta uma tarefa por ID
  deletar(req, res) {
    const sucesso = service.deletarTarefa(req.params.id);
    if (!sucesso) return res.status(404).json({ erro: 'Tarefa não encontrada.' });
    log('Tarefa deletada.');
    return res.status(204).send();
  },

  // Marca uma tarefa como concluída
  concluir(req, res) {
    const tarefa = service.concluirTarefa(req.params.id);
    if (!tarefa) return res.status(404).json({ erro: 'Tarefa não encontrada.' });
    log('Tarefa marcada como concluída.');
    return res.json(tarefa);
  }
};

// src/services/tarefasService.js

const { tarefas } = require('../database/fakeDb');
const { v4: uuidv4 } = require('uuid');

// Lógica de negócios
function adicionarTarefa(data) {
  const novaTarefa = { id: uuidv4(), ...data };
  tarefas.push(novaTarefa);
  return novaTarefa;
}

function listarTarefas(filtroConcluida) {
  return filtroConcluida !== undefined
    ? tarefas.filter(t => t.concluida === (filtroConcluida === 'true'))
    : tarefas;
}

function buscarPorId(id) {
  return tarefas.find(t => t.id === id);
}

function atualizarTarefa(id, data) {
  const index = tarefas.findIndex(t => t.id === id);
  if (index === -1) return null;
  tarefas[index] = { ...tarefas[index], ...data };
  return tarefas[index];
}

function deletarTarefa(id) {
  const index = tarefas.findIndex(t => t.id === id);
  if (index === -1) return false;
  tarefas.splice(index, 1);
  return true;
}

function concluirTarefa(id) {
  const tarefa = buscarPorId(id);
  if (!tarefa) return null;
  tarefa.concluida = true;
  return tarefa;
}

module.exports = {
  adicionarTarefa,
  listarTarefas,
  buscarPorId,
  atualizarTarefa,
  deletarTarefa,
  concluirTarefa
};

// src/middlewares/validateTarefa.js

const Joi = require('joi');

// Esquema de validação de tarefas
const tarefaSchema = Joi.object({
  titulo: Joi.string().min(3).required(),
  descricao: Joi.string().required(),
  concluida: Joi.boolean().required(),
});

// Middleware de validação
function validateTarefa(req, res, next) {
  const { error } = tarefaSchema.validate(req.body);
  if (error) {
    return res.status(400).json({ erro: error.details[0].message });
  }
  next();
}

module.exports = validateTarefa;

// src/utils/logger.js

// Função utilitária de logging simples
function log(message) {
  console.log(`[LOG] ${new Date().toISOString()} - ${message}`);
}

module.exports = { log };

// src/database/fakeDb.js

// Banco de dados falso em memória
const tarefas = [];
module.exports = { tarefas };
