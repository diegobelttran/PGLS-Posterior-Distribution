# PGLS-Posterior-Distribution
Comparing phenotypic traits with speciation/extinction rates using samples from a posterior distribution, with Phylogenetic Generalized Least Squares in R.

In my case I used color evolution rates obtained from BAMM as phenotypic traits and speciation rates obtained also from BAMM. BAMM is a Bayesian approach, hence, after running analyses you obtain a a posterior distributon of different trait evolution configurations.

What I aim to do here is to run a PGLS between color evolution rates and speciation rates. These rates will be averaged by clade. These clades will be chosen according to each BAMM configuration from a sample from the posterior distribution. So for example, if in one configuration form the posterior only one significant shift change was identified by BAMM, there will only be two clades with different rates. All tips within each clade will share their evolution rates. After averaging the rates of all tips within each clade I get two points in my color evolution rates vs. speciation rates plot, with these two points I run a PGLS and save the coefficients I get and repeat with the next sample from the posterior.

After repeating the same process with a good sample from the posterior, say 100, I get a sample of 100 PGLS coefficients and I can plot a histogram to see the general distributions of relationships between trait evolution and speciation, taking into account phylogenetic uncertainty.

This code is very rough and long I know, any help shortening and improving it is appreciated.

THIS SCRIPT IS STILL BEING TRANSLATED AND COMMENTED

##### BEGIN OF R SCRIPT

library(BAMMtools)
library(ape)
library(coda)
library(nlme)
library(scales)
library(phytools)
library(phangorn)


# phylogenetic trees as .tre files
tree <- read.tree('yourtree.tree')

# trees with species with data for both sexes, male data and female data, respectively
treeU <- read.tree('yourtreeU.tre')
tree_M <- read.tree('yourtreeM.tre')
tree_F <- read.tree('yourtreeF.tre')

# make a copy of thre tree just in case
copy <- tree

# Tabla de variables por especie y parche de ambos sexos
machos_prom <- read.table('~/Documents/Datos_colibries/BaseDeDatos/var_prom_sp_parche_machos.txt',h=T,sep='\t')
hembras_prom <- read.table('~/Documents/Datos_colibries/BaseDeDatos/var_prom_sp_parche_hembras.txt',h=T,sep='\t')

# tasas de especiacion de todas las especies en la filogenia completa
sp_r <- read.table('~/Documents/Datos_colibries/Speciation_rates/sp_rates_all.txt',h=T,sep='\t')

# tablas de tasas de evolucion de brillo y cromaticidad BAMM
sex_bri_mBeta <- read.table('~/Documents/Datos_colibries/Brightness_Beta/sex_bri_m_rates.txt',h=T,sep='\t')
rownames(sex_bri_mBeta) <- sex_bri_mBeta$Species
sex_bri_mBeta$Lambda <- sp_m$Lambda
nat_bri_mBeta <- read.table('~/Documents/Datos_colibries/Brightness_Beta/nat_bri_m_rates.txt',h=T,sep='\t')
rownames(nat_bri_mBeta) <- nat_bri_mBeta$Species
nat_bri_mBeta$Lambda <- sp_m$Lambda
sex_cromat_mBeta <- read.table('~/Documents/Datos_colibries/Chromaticity_Beta/sex_cromat_m_rates.txt',h=T,sep='\t')
rownames(sex_cromat_mBeta) <- sex_cromat_mBeta$Species
sex_cromat_mBeta$Lambda <- sp_m$Lambda
nat_cromat_mBeta <- read.table('~/Documents/Datos_colibries/Chromaticity_Beta/nat_cromat_m_rates.txt',h=T,sep='\t')
rownames(nat_cromat_mBeta) <- nat_cromat_mBeta$Species
nat_cromat_mBeta$Lambda <- sp_m$Lambda

sex_bri_fBeta <- read.table('~/Documents/Datos_colibries/Brightness_Beta/sex_bri_f_rates.txt',h=T,sep='\t')
rownames(sex_bri_fBeta) <- sex_bri_fBeta$Species
sex_bri_fBeta$Lambda <- sp_f$Lambda
nat_bri_fBeta <- read.table('~/Documents/Datos_colibries/Brightness_Beta/nat_bri_f_rates.txt',h=T,sep='\t')
rownames(nat_bri_fBeta) <- nat_bri_fBeta$Species
nat_bri_fBeta$Lambda <- sp_f$Lambda
sex_cromat_fBeta <- read.table('~/Documents/Datos_colibries/Chromaticity_Beta/sex_cromat_f_rates.txt',h=T,sep='\t')
rownames(sex_cromat_fBeta) <- sex_cromat_fBeta$Species
sex_cromat_fBeta$Lambda <- sp_f$Lambda
nat_cromat_fBeta <- read.table('~/Documents/Datos_colibries/Chromaticity_Beta/nat_cromat_f_rates.txt',h=T,sep='\t')
rownames(nat_cromat_fBeta) <- nat_cromat_fBeta$Species
nat_cromat_fBeta$Lambda <- sp_f$Lambda

# filtro las tasas de especiacion con las especies que tienen datos de cada sexo
sp_m <- sp_r[-which(is.na(match(sp_r$Species,sex_bri_mBeta$Species))),]
rownames(sp_m) <- sp_m$Species
sp_f <- sp_r[-which(is.na(match(sp_r$Species,sex_bri_fBeta$Species))),]
rownames(sp_f) <- sp_f$Species

#### tablas de tasa de evolucion de las variables de color de ambos sexos
#filtro las tablas de tasa de evolucion con las especies en comun de machos y hembras
sex_bri_mBeta <- sex_bri_mBeta[-which(is.na(match(sex_bri_mBeta$Species,sp_f$Species))),]
sex_bri_fBeta <- sex_bri_fBeta[-which(is.na(match(sex_bri_fBeta$Species,sp_m$Species))),]
sex_bri <- data.frame(Species=sex_bri_mBeta$Species,Lambda=sex_bri_mBeta$Lambda,BetaM=sex_bri_mBeta$Beta,BetaF=sex_bri_fBeta$Beta)
rownames(sex_bri) <- sex_bri$Species

nat_bri_mBeta <- nat_bri_mBeta[-which(is.na(match(nat_bri_mBeta$Species,sp_f$Species))),]
nat_bri_fBeta <- nat_bri_fBeta[-which(is.na(match(nat_bri_fBeta$Species,sp_m$Species))),]
nat_bri <- data.frame(Species=nat_bri_mBeta$Species,Lambda=nat_bri_mBeta$Lambda,BetaM=nat_bri_mBeta$Beta,BetaF=nat_bri_fBeta$Beta)
rownames(nat_bri) <- nat_bri$Species

sex_cromat_mBeta <- sex_cromat_mBeta[-which(is.na(match(sex_cromat_mBeta$Species,sp_f$Species))),]
sex_cromat_fBeta <- sex_cromat_fBeta[-which(is.na(match(sex_cromat_fBeta$Species,sp_m$Species))),]
sex_cromat <- data.frame(Species=sex_cromat_mBeta$Species,Lambda=sex_cromat_mBeta$Lambda,BetaM=sex_cromat_mBeta$Beta,BetaF=sex_cromat_fBeta$Beta)
rownames(sex_cromat) <- sex_cromat$Species

nat_cromat_mBeta <- nat_cromat_mBeta[-which(is.na(match(nat_cromat_mBeta$Species,sp_f$Species))),]
nat_cromat_fBeta <- nat_cromat_fBeta[-which(is.na(match(nat_cromat_fBeta$Species,sp_m$Species))),]
nat_cromat <- data.frame(Species=nat_cromat_mBeta$Species,Lambda=nat_cromat_mBeta$Lambda,BetaM=nat_cromat_mBeta$Beta,BetaF=nat_cromat_fBeta$Beta)
rownames(nat_cromat) <- nat_cromat$Species



# Agrego columna de nombre cientifico para hacerla coincidir con las terminales del arbol
tax <- read.table('~/Documents/Tax.txt',h=T,sep="\t")
tax$Nombre_cientifico <- gsub(pattern=' ',replacement='_',x=tax$Nombre_cientifico)
brilliants <- tax[which(tax$Clado == 'Brilliants'),]
bees <- tax[which(tax$Clado == 'Bees'),]
topazes <- tax[which(tax$Clado == 'Topazes'),]
hermits <- tax[which(tax$Clado == 'Hermits'),]
emeralds <- tax[which(tax$Clado == 'Emeralds'),]
gems <- tax[which(tax$Clado == 'Mtn_Gems'),]
coquettes <- tax[which(tax$Clado == 'Coquettes'),]
mangoes <- tax[which(tax$Clado == 'Mangoes'),]


machos_prom$Nombre_cientifico <- tax[match(machos_prom$Species,tax$ID_taxonomia),'Nombre_cientifico']
machos_prom$Nombre_cientifico <- gsub(pattern=' ',replacement='_',x=machos_prom$Nombre_cientifico)
hembras_prom$Nombre_cientifico <- tax[match(hembras_prom$Species,tax$ID_taxonomia),'Nombre_cientifico']
hembras_prom$Nombre_cientifico <- gsub(pattern=' ',replacement='_',x=hembras_prom$Nombre_cientifico)


copia2 <- makeNodeLabel(copia, method ='number', prefix='')
copia2$node.label <- 294:585
# motilar arbol para machos
cuales <- match(copia2$tip.label,unique(machos_prom$Nombre_cientifico))
drops <- which(is.na(cuales) == T)
copia_M <- drop.tip(copia2, drops)

# motilar arbol para hembras
cuales <- match(copia2$tip.label,unique(hembras_prom$Nombre_cientifico))
drops <- which(is.na(cuales) == T)
copia_F <- drop.tip(copia2, drops)

# motilar arbol unisex
cuales <- which(is.na(match(copia2$tip.label,sex_bri$Species))==T)
copia_U <- drop.tip(copia2,cuales)

# numero de los nodos de los 9 subclados
nodos <- c(296,299,329,356,401,446,449,463,490)

# tasas de especiacion
sp_r <- getEventData(arbol,'~/Documents/BAMM_Diversification/Sp_rates/event_sp_data.txt',burnin=0.1,type='diversification')
# tasas de evolucion de cada variable
sex_bri_m_r <- getEventData(arbol_M,'~/Documents/BAMM_Trait/Brightness/Males/Sex_patch/sex_bri_m_event_data.txt',burnin=0.1,type = "trait")
nat_bri_m_r <- getEventData(arbol_M,'~/Documents/BAMM_Trait/Brightness/Males/Nat_patch/nat_bri_m_event_data.txt',burnin=0.1,type = "trait")
sex_cromat_m_r <- getEventData(arbol_M,'~/Documents/BAMM_Trait/Chromaticity/Males/Sex_patch/sex_cromat_m_event_data.txt',burnin=0.1,type = "trait")
nat_cromat_m_r <- getEventData(arbol_M,'~/Documents/BAMM_Trait/Chromaticity/Males/Nat_patch/nat_cromat_m_event_data.txt',burnin=0.1,type = "trait")

sex_bri_f_r <- getEventData(arbol_F,'~/Documents/BAMM_Trait/Brightness/Females/Sex_patch/sex_bri_f_event_data20M-4k.txt',burnin=0.1,type = "trait")
nat_bri_f_r <- getEventData(arbol_F,'~/Documents/BAMM_Trait/Brightness/Females/Nat_patch/nat_bri_f_event_data30M-6k.txt',burnin=0.1,type = "trait")
sex_cromat_f_r <- getEventData(arbol_F,'~/Documents/BAMM_Trait/Chromaticity/Females/Sex_patch/sex_cromat_f_event_data.txt',burnin=0.1,type = "trait")
nat_cromat_f_r <- getEventData(arbol_F,'~/Documents/BAMM_Trait/Chromaticity/Females/Nat_patch/nat_cromat_f_event_data.txt',burnin=0.1,type = "trait")


#tomo 100 muestras de la distribucion posterior de tasas de evolucion y de especiacion
sp_r_sub1 <- sp_r[sample(1000)]
sex_bri_m_r_sub1 <- sex_bri_m_r[sample(1000)]
sex_bri_f_r_sub1 <- sex_bri_f_r[sample(1000)]

#creo dataframe donde iran los coeficientes de correlacion de los PGLS de cada mutesra de la distribucion posterior
coef_sex_bri <- data.frame(matrix(NA,ncol=7))
names(coef_sex_bri) <- c('Intercept','Male_slope','Female_slope','p-value_Intercept','p-value_Male','p-value_Female','Number_clades')

#bucle para obtener coeficiencites de PGLS para 100 muestras de la distribucion posterior
for (u in 1:1000){

#creo un data frame con los nodos del arbol asignados por bamm en el arbol de machos y los del arbol original completo
nodos_1 <- data.frame(nodo.actual= 257:(256+255), nodo.orig=copia_M$node.label)
#obtengo los cambios que ocurrieron en terminales, es decir los cambios en 'nodos' menores a 257 que es el primer nodo del arbol cortado
cuales <- which(sex_bri_m_r_sub1$eventData[[u]]$node < 257)
#nodos donde ocurrieron cambios que no son terminales
cuales.no <- which(sex_bri_m_r_sub1$eventData[[u]]$node >= 257)
#especies donde ocurrio el cambio en el arbol que se analizo en bamm
especies.orig <- copia_M$tip.label[sex_bri_m_r_sub1$eventData[[u]]$node[cuales]]
#especies donde ocurrio el cambio en el arbol completo
especies.comp <- match(especies.orig, copia2$tip.label)
#nuevo vector con nodos y terminales segun arbol completo
v.final <- v.final.0 <- sex_bri_m_r_sub1$eventData[[u]]$node
v.final[cuales] <- especies.comp
v.final[cuales.no] <- nodos_1[match(v.final.0[cuales.no],nodos_1[,1]),2]


#creo un data frame con los nodos del arbol asignados por bamm y los del arbol original completo
nodos_1F <- data.frame(nodo.actual= (length(copia_F$tip.label)+1):(length(copia_F$tip.label)+length(copia_F$tip.label)-1), nodo.orig=copia_F$node.label)
#obtengo los cambios que ocurrieron en terminales, es decir los cambios en 'nodos' menores a 257 que es el primer nodo del arbol cortado
cualesF <- which(sex_bri_f_r_sub1$eventData[[u]]$node < length(copia_F$tip.label)+1)
#nodos donde ocurrieron cambios que no son terminales
cuales.noF <- which(sex_bri_f_r_sub1$eventData[[u]]$node >= length(copia_F$tip.label)+1)
#especies donde ocurrio el cambio en el arbol que se analizo en bamm
especies.origF <- copia_F$tip.label[sex_bri_f_r_sub1$eventData[[u]]$node[cualesF]]
#especies donde ocurrio el cambio en el arbol completo
especies.compF <- match(especies.origF, copia2$tip.label)
#nuevo vector con nodos y terminales segun arbol completo
v.finalF <- v.final.0F <- sex_bri_f_r_sub1$eventData[[u]]$node
v.finalF[cualesF] <- especies.compF
v.finalF[cuales.noF] <- nodos_1F[match(v.final.0F[cuales.noF],nodos_1F[,1]),2]




#agregar cambios de especiacion y clados, obteniendo solo los clados unicos, sin repeticiones
v.final2 <- unique(c(v.final,v.finalF,sp_r_sub1$eventData[[u]]$node,nodos))

#creo un data frame con los nodos del arbol asignados por bamm y los del arbol original completo
nodos_2 <- data.frame(nodo.actual= 238:(237+236), nodo.orig=copia_U$node.label)
#obtengo los numeros de las ramas donde hubo cambios
cuales <- which(v.final2 < 294)
#obtengo los numeros de los nodos donde hubo cambios
cuales.no <- which(v.final2 >= 294)
#especies donde ocurrieron cambios segun el arbol completo
cuales.sp <- copia2$tip.label[v.final2[cuales]]
#especies donde ocurrieron cambios en el arbol unisex
esp.final <- na.omit(match(cuales.sp, copia_U$tip.label))
#nodos donde ocurrieron cambios en el arbol unisex
nod.final <- na.omit(nodos_2[match(v.final2[cuales.no],nodos_2[,2]),1])
#nuevo vector con las especies y nodos donde ocurrieron cambios en el arbol unisex
v.uni <- c(esp.final,nod.final)

#se obtienen los nodos y terminales descendientes de todos los nodos
des <- Descendants(copia_U,node=v.uni,type='all')

#creo una tabla con el numero de nodos o terminales de v.uni que estan contenidos en otro
nodos_c <- data.frame(matrix(NA,ncol=2,nrow=length(v.uni)))
names(nodos_c) <- c('Nodo','Nodos_contenidos')
rep <- lapply(des,FUN=function(x) match(v.uni,x))
nodos_c[,1] <- v.uni
nodos_c[,2] <- sapply(rep, FUN=function(x) length(na.omit(x)))

#lista con las terminales de cada uno de los nodos en nodos_c
terminales <- Descendants(copia_U,node=v.uni,type='tips')

###
terminal.final <- terminales
#cuales nos interesa modificar. Nodos con al menos una terminal o un nodo en su interior.
cuales.nodos <- nodos_c[nodos_c[,1] > 237 & nodos_c[,2] > 0 ,1]

for (i in 1:length(cuales.nodos)) {
pos <- match(cuales.nodos, nodos_c[,1])

#nodos y terminales que contiene el nodo i
nodos.cont <- des[[pos[i]]][na.omit(rep[[pos[i]]])]
#terminales que contienen todos esos nodos
tip.menos <- unique(do.call(c,terminales[match(nodos.cont, nodos_c[,1])]))
#terminales que contiene nodo i
tip.i <- terminales[[pos[i]]]
#verificar si nodo se elimina (todas las terminales estan contenidas en otros nodos)
if (length(is.na(match(tip.menos, tip.i)) == FALSE) == length(tip.i)) { 
terminal.final[[pos[i]]] <- 0
#si Verdadero, se elimina nodo
} else {  
tip.i.fin <- tip.i[-(match(tip.menos, tip.i))]
terminal.final[[pos[i]]] <- tip.i.fin
}
}

if (length(is.na(match(tip.menos, tip.i)) == FALSE) == length(tip.i)) terminal.final[[pos[i]]] <- NULL

#creo un indice de los nodos que no son vacios
indice <- which(do.call(c,lapply(terminal.final,function(x) (x[1] == 0))) == FALSE)
#cuales nodos
nodos.graf <- do.call(c,(terminal.final[indice]))
#cuantos nodos
cuan.nodo <- do.call(c,lapply(terminal.final[indice], length))
#cols <- rainbow(length(cuan.nodo))
#colores <- rep(cols,cuan.nodo)
#plot(copia_U, show.tip.label=FALSE)
#nodelabels(node=nodos.graf,frame='none', cex=0.5, col=colores)


####obtener los nombres de las terminales finales
nombres.final <- terminal.final
for (i in 1:length(terminal.final)){
if(length(terminal.final[[i]]) == 0){
next} else {
nombres.final[[i]] <- copia_U$tip.label[nombres.final[[i]]]
}
}

#creo el vector que va a agrupar las especies en los nodos que se establecieron
grupos <- rep(1:length(indice),cuan.nodo)
#reordeno la tabla sex_bri para que quede en el orden de nombres.final usando los rownames
sex_bri_grupos <- sex_bri[do.call(c,nombres.final),]
#obtengo los promedios por nodo
betaM <- tapply(sex_bri_grupos$BetaM, INDEX=grupos, FUN=mean)
betaF <- tapply(sex_bri_grupos$BetaF, INDEX=grupos, FUN=mean)
lambda <- tapply(sex_bri_grupos$Lambda, INDEX=grupos, FUN=mean)
sex_briC <- data.frame(Lambda=lambda,BetaM=betaM,BetaF=betaF)
#agrego los rownames de las especies que van a representar cada uno de los nodos (clados)
rownames(sex_briC ) <- rownames(sex_bri_grupos)[match(1:length(indice),grupos)]


#estos son las especies que son diferentes a las representantes y se van a cortar del arbol
podar <- rownames(sex_bri_grupos)[(1:length(grupos))[-(match(1:length(indice),grupos))]]
#arbol solo con las ramas representantes
subtre1 <- drop.tip(copia_U,podar)
#pgls aditivo con los dos sexos
pgls_sex_bri <- gls(Lambda ~ BetaM + BetaF ,data=sex_briC,correlation= corBrownian(phy=subtre1), method="REML")
summ <- as.data.frame(summary(pgls_sex_bri)$tTable)
coef(pgls_sex_bri)
#plot(Lambda ~ BetaM,data=sex_briC,pch=16,xlab="Brightness evolution rates",ylab="Speciation rates",ylim=c(0.15,0.35),cex=1.6,main='Sexual selection patches',cex.lab=1.4,cex.axis=1.4,cex.main=1.5)


#lleno la tabla coef_sex_bri con los coeficientes de cada iteracion/muestra de la distribucion posterior
coef_sex_bri[u,1:3] <- t(summ[,1])
coef_sex_bri[u,4:6] <- t(summ[,4])
coef_sex_bri[u,'Number_clades'] <- length(unique(grupos))
}

hist(coef_sex_bri$Male_slope)
abline(v=0.53349536,lwd=2,col='red')
quantile(coef_sex_bri$Male_slope,c(0.05,0.95))
0.53349536 > quantile(coef_sex_bri$Male_slope,c(0.05,0.95))[2]
# 95% 
#TRUE 

hist(coef_sex_bri[which(coef_sex_bri[,5] <0.05),'Male_slope'],main='Histogram of 1000 PGLS regression coefficients',xlab='Significant Male slope coefficients',cex.lab=1.45,cex.axis=1.3)
