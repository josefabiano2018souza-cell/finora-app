package com.example.finora

import android.content.Context
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import androidx.room.*
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch

// ==========================================
// 1. BANCO DE DADOS (ROOM)
// ==========================================

@Entity(tableName = "transactions")
data class TransactionEntity(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val title: String,
    val amount: Double,
    val category: String,
    val isIncome: Boolean,
    val date: Long = System.currentTimeMillis()
)

@Dao
interface TransactionDao {
    @Query("SELECT * FROM transactions ORDER BY date DESC")
    fun getAllTransactions(): kotlinx.coroutines.flow.Flow<List<TransactionEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertTransaction(transaction: TransactionEntity)

    @Delete
    suspend fun deleteTransaction(transaction: TransactionEntity)
}

@Database(entities = [TransactionEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun transactionDao(): TransactionDao

    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getDatabase(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "finora_db"
                ).build()
                INSTANCE = instance
                instance
            }
        }
    }
}

// ==========================================
// 2. VIEWMODEL
// ==========================================

class FinoraViewModel(private val dao: TransactionDao) : ViewModel() {
    val transactions: StateFlow<List<TransactionEntity>> = dao.getAllTransactions()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun addTransaction(title: String, amount: Double, category: String, isIncome: Boolean) {
        viewModelScope.launch {
            dao.insertTransaction(
                TransactionEntity(
                    title = title,
                    amount = amount,
                    category = category,
                    isIncome = isIncome
                )
            )
        }
    }

    fun deleteTransaction(transaction: TransactionEntity) {
        viewModelScope.launch {
            dao.deleteTransaction(transaction)
        }
    }
}

// ==========================================
// 3. TELA E COMPONENTES (COMPOSE)
// ==========================================

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val db = AppDatabase.getDatabase(this)
        val viewModel = FinoraViewModel(db.transactionDao())

        setContent {
            MaterialTheme {
                Surface(modifier = Modifier.fillMaxSize()) {
                    FinoraScreen(viewModel)
                }
            }
        }
    }
}

@Composable
fun FinoraScreen(viewModel: FinoraViewModel) {
    val transactions by viewModel.transactions.collectAsState()
    var title by remember { mutableStateOf("") }
    var amount by remember { mutableStateOf("") }
    var isIncome by remember { mutableStateOf(false) }

    val totalBalance = transactions.sumOf { if (it.isIncome) it.amount else -it.amount }

    Column(modifier = Modifier.fillMaxSize().padding(16.dp)) {
        Text("Finora - Controle Financeiro", fontSize = 24.sp, fontWeight = FontWeight.Bold)
        Spacer(modifier = Modifier.height(8.dp))
        
        Card(modifier = Modifier.fillMaxWidth().padding(vertical = 8.dp)) {
            Column(modifier = Modifier.padding(16.dp)) {
                Text("Saldo Total", fontSize = 14.sp)
                Text(
                    text = "R$ %.2f".format(totalBalance),
                    fontSize = 28.sp,
                    fontWeight = FontWeight.Bold,
                    color = if (totalBalance >= 0) Color(0xFF2E7D32) else Color(0xFFC62828)
                )
            }
        }

        Spacer(modifier = Modifier.height(16.dp))

        OutlinedTextField(
            value = title,
            onValueChange = { title = it },
            label = { Text("Descrição") },
            modifier = Modifier.fillMaxWidth()
        )
        
        OutlinedTextField(
            value = amount,
            onValueChange = { amount = it },
            label = { Text("Valor (R$)") },
            modifier = Modifier.fillMaxWidth()
        )

        Row(
            verticalAlignment = Alignment.CenterVertically,
            modifier = Modifier.padding(vertical = 8.dp)
        ) {
            RadioButton(selected = !isIncome, onClick = { isIncome = false })
            Text("Despesa")
            Spacer(modifier = Modifier.width(16.dp))
            RadioButton(selected = isIncome, onClick = { isIncome = true })
            Text("Receita")
        }

        Button(
            onClick = {
                val parsedAmount = amount.toDoubleOrNull() ?: 0.0
                if (title.isNotBlank() && parsedAmount > 0) {
                    viewModel.addTransaction(title, parsedAmount, "Geral", isIncome)
                    title = ""
                    amount = ""
                }
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("Adicionar Transação")
        }

        Spacer(modifier = Modifier.height(16.dp))

        Text("Histórico", fontSize = 18.sp, fontWeight = FontWeight.Bold)

        LazyColumn {
            items(transactions) { item ->
                Card(modifier = Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
                    Row(
                        modifier = Modifier.padding(16.dp).fillMaxWidth(),
                        horizontalArrangement = Arrangement.SpaceBetween
                    ) {
                        Column {
                            Text(item.title, fontWeight = FontWeight.Bold)
                            Text(if (item.isIncome) "Receita" else "Despesa", fontSize = 12.sp)
                        }
                        Text(
                            text = (if (item.isIncome) "+ R$ " else "- R$ ") + "%.2f".format(item.amount),
                            color = if (item.isIncome) Color(0xFF2E7D32) else Color(0xFFC62828),
                            fontWeight = FontWeight.Bold
                        )
                    }
                }
            }
        }
    }
}
