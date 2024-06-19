//+------------------------------------------------------------------+
//|                                                  candlestick.mq5 |
//|                                  Copyright 2024, MetaQuotes Ltd. |
//|                                             https://www.mql5.com |
//+------------------------------------------------------------------+
#property copyright "Copyright 2024, MetaQuotes Ltd."
#property link      "https://www.mql5.com"
#property version   "1.00"
//+------------------------------------------------------------------+
//| defines                                                          |
//+------------------------------------------------------------------+
#define NR_CONDITIONS 2       //numero de condiciones

//+------------------------------------------------------------------+
//| include                                                          |
//+------------------------------------------------------------------+
#include <Trade\Trade.mqh>

//+------------------------------------------------------------------+
//| variables globales                                               |
//+------------------------------------------------------------------+
enum MODE{
   OPEN=0,     // abrir
   HIGH=1,     // alto
   LOW=2,      // bajo
   CLOSE=3,    // cierre
   RANGE=4,    // rango (puntos)
   BODY=5,     // cuerpo (puntos)
   RATIO=6,    // ratio
   VALUE=7     // valor
};

enum INDEX{
   INDEX_0=0,  //index 0
   INDEX_1=1,  //index 1
   INDEX_2=2,  //index 2
   INDEX_3=3,  //index 3
};

enum COMPARE{
   GREATER,    // mayor que
   LESS        // menos
};

struct CONDITION{
   bool active;   // condicion activa ?
   MODE modeA;    // modo A
   INDEX idxA;    // index A
   COMPARE comp;  // comparar
   MODE modeB;    // modo B
   INDEX idxB;    // index B
   double value;  // valor
   
   CONDITION () : active(false){};
};

CONDITION con[NR_CONDITIONS]; // matriz de condiciones
MqlTick currentTick;          // current del simbolo actual
CTrade trade;                 // objeto de abrir/cerrar posiciones

//+------------------------------------------------------------------+
//| Expert initialization function                                   |
//+------------------------------------------------------------------+
input group "==== general ====";
static input long    InpMagicNumber = 762343;  // numero magico
static input double  InpLots        = 0.01;    // lotaje
input  int           InpStopLoss    = 100;     // stop loss
input  int           InpTakeProfit  = 200;     // take profit

input group "==== condicion 1 ====";
input bool           InpCon1Active  = true;    // activo
input MODE           InpCon1ModeA   = OPEN;    // modo A
input INDEX          InpCon1IndexA  = INDEX_1; // index A
input COMPARE        InpCon1Compare = GREATER; // comparar
input MODE           InpCon1ModeB   = CLOSE;   // modo B
input INDEX          InpCon1IndexB  = INDEX_1; // index B
input double         InpCon1Value   = 0;       // validar

input group "==== condicion 2 ====";
input bool           InpCon2Active  = false;   // activo
input MODE           InpCon2ModeA   = OPEN;    // modo A
input INDEX          InpCon2IndexA  = INDEX_1; // index A
input COMPARE        InpCon2Compare = GREATER; // comparar
input MODE           InpCon2ModeB   = CLOSE;   // modo B
input INDEX          InpCon2IndexB  = INDEX_1; // index B
input double         InpCon2Value   = 0;       // validar 
//+------------------------------------------------------------------+
//| Expert initialization function                                   |
//+------------------------------------------------------------------+
int OnInit()
  {
   
   // verificar las entradas de las entradas
   SetInputs();
   
   // verificar las entradas
   if(!CheckInputs ()){return INIT_PARAMETERS_INCORRECT;}
   
   //obtener el numero magico
   trade.SetExpertMagicNumber(InpMagicNumber);
   
   return(INIT_SUCCEEDED);
  }
//+------------------------------------------------------------------+
//| Expert deinitialization function                                 |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
  {

   
  }
//+------------------------------------------------------------------+
//| Expert tick function                                             |
//+------------------------------------------------------------------+
void OnTick()
  {
      // verificar si el actual tickeck es una nueva barra abierta 
      if(!IsNewBar()){return;}
   
      // conseguir verificar el simbolo tick
      if(!SymbolInfoTick(_Symbol,currentTick)){Print("fallo en conseguir el tikect actual"); return;}
      
      // contar las posicioens abiertas
      int cntBuy, cntSell;
      if(!CountOpenPositions(cntBuy,cntSell)){Print("fallo en obtener el conteo de posiciones abiertos"); return;}
      
      // verifica si hay nuevas posiciones abiertas
      if(cntBuy==0 && CheckAllConditions(true)){
      
         // calcula stop loss y take profit
         double sl = InpStopLoss==0 ? 0 : currentTick.bid - InpStopLoss *_Point;
         double tp = InpTakeProfit==0 ? 0 : currentTick.bid + InpTakeProfit *_Point;
         if(!NormalizePrice (sl)){return;}
         if(!NormalizePrice (tp)){return;}
      
         trade.PositionOpen(_Symbol,ORDER_TYPE_BUY,InpLots,currentTick.ask,sl,tp,"Patron De Velas EA");
      }
      
      // verifica si hay nuevas posiciones abiertas
      if(cntSell==0 && CheckAllConditions(false)){
      
         // calcula stop loss y take profit
         double sl = InpStopLoss==0 ? 0 : currentTick.ask + InpStopLoss *_Point;
         double tp = InpTakeProfit==0 ? 0 : currentTick.ask - InpTakeProfit *_Point;
         if(!NormalizePrice (sl)){return;}
         if(!NormalizePrice (tp)){return;}
      
         trade.PositionOpen(_Symbol,ORDER_TYPE_SELL,InpLots,currentTick.bid,sl,tp,"Patron De Velas EA");
      }
  }

//+------------------------------------------------------------------+
//| funciones personalizadas                                         |
//+------------------------------------------------------------------+

void SetInputs (){

   // condicion 1
   con [0] .active = InpCon1Active;
   con [0] .modeA  = InpCon1ModeA;
   con [0] .idxA   = InpCon1IndexA;
   con [0] .comp   = InpCon1Compare;
   con [0] .modeB  = InpCon1ModeB;
   con [0] .idxB   = InpCon1IndexB;
   con [0] .value  = InpCon1Value;
   
    // condicion 2
   con [1] .active = InpCon2Active;
   con [1] .modeA  = InpCon2ModeA;
   con [1] .idxA   = InpCon2IndexA;
   con [1] .comp   = InpCon2Compare;
   con [1] .modeB  = InpCon2ModeB;
   con [1] .idxB   = InpCon2IndexB;
   con [1] .value  = InpCon2Value;
}

bool CheckInputs (){

   if(InpMagicNumber <=0){
      Alert("entrada incorrecta: numero magico <=0 ");
      return false;
   }
   if(InpLots <=0){
      Alert("entrada incorrecta: lotes <=0 ");
      return false;
   }
   if(InpStopLoss <0){
      Alert("entrada incorrecta: stop loss <=0 ");
      return false;
   }
   if(InpTakeProfit <0){
      Alert("entrada incorrecta: take profit <=0 ");
      return false;
   }

   // verificamos las condiciones +++
   
   
   return true;
}

bool CheckAllConditions(bool buy_sell)
{
   //comprobar cada condicion
   for(int i=0; i<NR_CONDITIONS; i++){
      if(!CheckOneCondition(buy_sell,i)){return false;}
   }

   return true;
}

bool CheckOneCondition(bool buy_sell, int i)
{
   //retorno verdadero si la condicion no esta activa 
   if(!con[i].active){return true;}
   
   // obetener barra data
   MqlRates rates[];
   ArraySetAsSeries(rates,true);
   int copied = CopyRates(_Symbol,PERIOD_CURRENT,0,4,rates);
   if(copied!=4){
      Print("fallo en obtener la barra de la data. copiada:",(string)copied);
      return false;
   }
   
   // establecer valores de a y  b
   double a=0;
   double b=0;
   
   switch(con[i].modeA){
      case OPEN:  a = rates[con[i].idxA].open; break;
      case HIGH:  a = buy_sell ? rates[con[i].idxA].high : rates[con[i].idxA].low; break;
      case LOW:   a = buy_sell ? rates[con[i].idxA].low : rates[con[i].idxA].high; break;
      case CLOSE: a = rates[con[i].idxA].close; break;
      case RANGE: a = (rates[con[i].idxA].high - rates[con[i].idxA].low) / _Point; break;
      case BODY:  a = MathAbs(rates[con[i].idxA].open - rates[con[i].idxA].close) / _Point; break;
      case RATIO: a = MathAbs(rates[con[i].idxA].open - rates[con[i].idxA].close) /
                        (rates[con[i].idxA].high - rates[con[i].idxA].low); break;
      case VALUE: a = con[i].value; break;
      default:    return false;
   }
   switch(con[i].modeB){
      case OPEN:  b = rates[con[i].idxB].open; break;
      case HIGH:  b = buy_sell ? rates[con[i].idxB].high : rates[con[i].idxB].low; break;
      case LOW:   b = buy_sell ? rates[con[i].idxB].low : rates[con[i].idxB].high; break;
      case CLOSE: b = rates[con[i].idxB].close; break;
      case RANGE: b = (rates[con[i].idxB].high - rates[con[i].idxB].low) / _Point; break;
      case BODY:  b = MathAbs(rates[con[i].idxB].open - rates[con[i].idxB].close) / _Point; break;
      case RATIO: b = MathAbs(rates[con[i].idxB].open - rates[con[i].idxB].close) /
                        (rates[con[i].idxB].high - rates[con[i].idxB].low); break;
      case VALUE: b = con[i].value; break;
      default:    return false;
   }
   
   // comparar validacion
   if(buy_sell || (!buy_sell && con[i].modeA>=4)){
      if(con[i].comp==GREATER && a>b){return true;}
      if(con[i].comp==LESS && a<b){return true;}
      
   }else{
      if(con[i].comp==GREATER && a<b){return true;}
      if(con[i].comp==LESS && a>b){return true;}
   }

   return false;
}

//verificamos si tenemos una barra  abierta en el tikect
bool IsNewBar(){

   static datetime previousTime = 0;
   datetime currentTime = iTime(_Symbol,PERIOD_CURRENT,0);
   if(previousTime!=currentTime){
      previousTime=currentTime;
      return true;
   }
   return false;

}

// contar posicion abierta
bool CountOpenPositions(int &cntBuy, int &cntSell){
   cntBuy= 0;
   cntSell = 0;
   int total = PositionsTotal();
   for(int i=total-1; i>=0; i--){
      ulong ticket = PositionGetTicket(i);
      if(ticket<=0){Print("No se pudo conseguir la posicion"); return false;}
      if(!PositionSelectByTicket(ticket)){Print("no se pudo seleccionar la posición"); return false;}
      long magic;
      if(!PositionGetInteger(POSITION_MAGIC,magic)){Print("No se pudo obtener la posición del número mágico"); return false;}
      if(magic==InpMagicNumber){
         long type;
         if(!PositionGetInteger(POSITION_TYPE,type)){Print("no se pudo obtener el tipo de posición"); return false;}
         if(type==POSITION_TYPE_BUY){cntBuy++;}
         if(type==POSITION_TYPE_SELL){cntSell++;}
      }
   }
   
   return true;
}

// normalizar precio 
bool NormalizePrice(double &price){
   
   double tickSize =0;
   if(!SymbolInfoDouble(_Symbol,SYMBOL_TRADE_TICK_SIZE,tickSize)){
      Print("fallo en obtener el ticket");
      return false;
   }
   price = NormalizeDouble(MathRound(price/tickSize)*tickSize,_Digits);
   
   return true;
}
