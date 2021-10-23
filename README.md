## Fundamentos de testes automatizados

### Tipos de testes

#### Unitário

<p>Teste feito pelo desenvolvedor, responsável por validar o comportamento de unidades funcionais de código. Nesse contexto, entende-se como unidade funcional qualquer porção de código que através de algum estímulo seja capaz de gerar um comportamento esperado (na prática: métodos de uma classe). Um teste unitário não pode acessar outros componentes ou recursos externos (arquivos, bd, rede, web services, etc.).</p>

#### Integração

<p>Teste focado em verificar se a comunicação entre componentes / módulos da aplicação, e também recursos externos, estão interagindo entre si corretamente.</p>

### Funcional

<p>É um teste do ponto de vista do usuário, se uma determinada funcionalidade está executando corretamente, produzindo o resultado ou comportamento desejado pelo usuário.</p>

#### Beneficios

-	Detectar facilmente se mudanças violaram as regras
-	É uma forma de documentação (comportamento e entradas/saídas esperadas)
-	Redução de custos em manutenções, especialmente em fases avançadas
-	Melhora design da solução, pois a aplicação testável precisa ser bem delineada


## Refactoring

> dscatalog-backend1-main-cap1<br />
> dscatalog-backend1-main-cap1 [boot] [dscatalog-backend1-main-cap1   master]

### Annotations usadas nas classes de teste

| Annotation | Objectivo |
|-----------------|-------------------------------------------------------|
| @SpringBootTest | Carrega o contexto da aplicação (teste de integração) |
| @SpringBootTest<br />@AutoConfigureMockMvc | Carrega o contexto da aplicação (teste de integração & web) <br /> Trata as requisições sem subir o servidor|
| @WebMvcTest(Classe.class) | Carrega o contexto, porém somente da camada web (teste de unidade: controlador) |
| @ExtendWith(SpringExtension.class) | Não carrega o contexto, mas permite usar os recursos do Spring com JUnit (teste de unidade: service/component) |
| @DataJpaTest | Carrega somente os componentes relacionados ao Spring Data JPA. <br />Cada teste é transacional e dá rollback ao final. (teste de unidade: repository) |


## Fixtures
É uma forma de organizar melhor o código dos testes e evitar repetições.

| JUnit5 |      JUnit4     | Objectivo          |
|--------|-----------------|--------------------|
| @BeforeAll| @BeforeClass | Preparação antes de todos testes da classe (método estático)|
| @AfterAll | @AfterClass | Preparação depois de todos testes da classe (método estático) |
| @BeforeEach | @Before | Preparação antes de cada teste da classe  |
| @AfterEach | @After | Preparação depois de cada teste da classe |

### Teste ao método save() com id nulo
Teste ao `save()` do repository para ver se está a guardar um novo objecto quando o **id** é nulo.
No `ProductService` os métodos para criar e atualizar produtos - `insert()` e `update()`, utilizam os dois o método `save()`. A diferença é que quando inserimos um novo objecto o **id** é nulo, no entanto quando vai fazer o `update()` o `save()` tem que usar um **id** que já existe na **BD**.

```java
@Test
public void saveShouldPersistWithAutoincrementWhenIdIsNull() {	
	Product product = new Product();
}
```
**Criação de classe auxiliar `Factory` no package de testes:** <br />
**Package:** `com.devsuperior.dscatalog.tests`

As classes *Factory* são responsáveis por instanciar objectos. Neste caso, criação de método estático para retornar um novo produto.

```java
public class Factory {

	public static Product createProduct() {
		Product product = new Product(1L, "Livros", "Demon Slayer N.º 01", 8.99, "https://www.wook.pt/livro/demon-slayer-n-01-koyoharu-gotouge/24733957", Instant.parse("2021-10-30T18:35:24.00Z"));
		product.getCategories().add(new Category(1L, "Livros"));
		return product;
	}
}
```

Criação de um método static para um `ProductDTO`. Aproveitamos a construção do `createProduct()` e passamos ao construtor do `ProductDTO`.

```java
public static ProductDTO createProductDTO() {
	Product product = createProduct();
	return new ProductDTO(product, product.getCategories());
}
```
Na classe `ProductDTO` temos o construtor `ProductDTO`, o qual instancia uma lista de categorias:

```java
public ProductDTO(Product entity, Set<Category> categories) {
	this(entity);
	categories.forEach(cat -> this.categories.add(new CategoryDTO(cat)));
}
```

Na classe `ProductRepository` garantimos que o **id** vai ser nulo:

```java
@Test
public void saveShouldPersistWithAutoincrementWhenIdIsNull() {		
	Product product = Factory.createProduct();
	product.setId(null);
}
```
Testamos através do `product.getId()` se o **id** não é nulo:
```java
@DataJpaTest
public class ProductRepositoryTests {
    (...)

    @Test
	public void saveShouldPersistWithAutoincrementWhenIdIsNull() {		
		
        (...)
		
		Assertions.assertNotNull(product.getId());
	}
```

Testamos se o valor do **id** vai ser o 26, visto existirem 25 produtos na **BD**:

```java
@DataJpaTest
public class ProductRepositoryTests {

	(...)

	private long countTotalProducts;
	
	@BeforeEach
	void setUp() throws Exception {
		(...)
		countTotalProducts = 56L;
	}
	
	@Test
	public void saveShouldPersistWithAutoincrementWhenIdIsNull() {				
        (...)
		Assertions.assertEquals(countTotalProducts + 1, product.getId());
	}
```
#### Exercícios: testes de repository
**Solução:** https://youtu.be/qm3K1dkzJBM

Implementar os seguintes testes em `ProductRepositoryTests`:
-	`findById` deveria 
    -	retornar um `Optional<Product>` não vazio quando o id existir
    -	retornar um `Optional<Product>` vazio quando o id não existir

Retornar um `Optional<Product>` não vazio quando o id existir:

```java
@Test
public void findByIdShouldReturnNonEmptyOptionalProductWhenIdExists() {
    
    Optional<Product> result = repository.findById(existingId);
    Assertions.assertTrue(result.isPresent());
    
}
```
Retornar um `Optional<Product>` vazio quando o id não existir:

```java
@Test
public void findByIdShouldReturnAnEmptyOptionalProductWhenIdDoesNotExists() {
            
    Optional<Product> result = repository.findById(longNonExistingId);
    Assertions.assertTrue(result.isEmpty());
    
}
```
## Começando testes de ProductService, Mockito vs MockBean
### Testes de unidade `ProductService`
Trabalhar com testes de unidade implica trabalhar sem carregar outros componentes de que a classe depende, como por exemplo o  `ProductRepository` ou o `CategoryRepository`.<br/>
Para se poder testar a classe `ProductService` temos que *Mockar* as dependências, ou seja, temos de usar o **Mockito**. Ao carregar outros componentes significa que já não seriam testes unitários mas sim testes de integração. Estaríamos a testar a integração do `ProductService` com o `ProductRepository` ou o `CategoryRepository`. Os testes de unidade são isolados e mais rápidos, são importantes para validar um componente específico.
Os testes de integração ou testes que tenham que carregar contextos mais complicados no *spring* não costumam ser feitos com muita regularidade.

```java
@ExtendWith(SpringExtension.class)
public class ProductServiceTests {
    
    @InjectMocks
	private ProductService service;
}
```
Não utilizamos o *@Autowired* para não injectar o service normal mas sim uma notation especial designada de *@InjectMocks*.
Sempre que precisarmos de utilizar neste caso o `ProductRepository` ou o `CategoryRepository` temos que simular os seus comportamentos, temos que usar um objecto *Mockado* para simular os seus comportamentos.

Uma das forma mais usuais de usar um objecto *Mockado* é a seguinte:

```java
@Mock
private ProductRepository repository;
```

```java
package com.devsuperior.dscatalog.services;

import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import com.devsuperior.dscatalog.repositories.ProductRepository;

@ExtendWith(SpringExtension.class)
public class ProductServiceTests {

	@InjectMocks
	private ProductService service;
	
	@Mock
	private ProductRepository repository;
}
```

Neste exemplo verificamos que está a ser usado nas dependências seguintes:

```java
import org.mockito.InjectMocks;
import org.mockito.Mock;
```
Estas dependências são do **Mockito Vanilla** normal. Existe também a notation *@MockBean*

```java
import org.springframework.boot.test.mock.mockito.MockBean;
```
```java
@MockBean
private ProductRepository repository2;
```
```java
package com.devsuperior.dscatalog.services;

import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import com.devsuperior.dscatalog.repositories.ProductRepository;

@ExtendWith(SpringExtension.class)
public class ProductServiceTests {

	@InjectMocks
	private ProductService service;
	
	@Mock
	private ProductRepository repository;
	
	@MockBean
	private ProductRepository repository2;
}
```
O *MockBean* está empacotado dentro de um *mockito* no *spring* e os mesmos estão declarados no ficheiro **pom.xml**.

# Mockito vs @MockBean

https://stackoverflow.com/questions/44200720/difference-between-mock-mockbean-and-mockito-mock

| Annotation           | Designação |
|----------------------|------------|
| @Mock <br />private MyComp myComp; <br /> ou <br /> myComp=Mockito.mock(MyComp.class); | Usar quando a classe de teste não carrega o contexto da aplicação. É mais rápido e enxuto<br />@ExtendWith |
| @MockBean <br /> private MyComp myComp; | Usar quando a classe de teste carrega o contexto da aplicação e precisa mockar algum bean do sistema. <br /> @WebMvcTest <br /> @SpringBootTest |

## Primeiro teste, simulando comportamento com Mockito

#### Classe `ProductService`
```java
package com.devsuperior.dscatalog.services;

import java.util.Optional;

import javax.persistence.EntityNotFoundException;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.dao.EmptyResultDataAccessException;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.devsuperior.dscatalog.dto.CategoryDTO;
import com.devsuperior.dscatalog.dto.ProductDTO;
import com.devsuperior.dscatalog.entities.Category;
import com.devsuperior.dscatalog.entities.Product;
import com.devsuperior.dscatalog.repositories.CategoryRepository;
import com.devsuperior.dscatalog.repositories.ProductRepository;
import com.devsuperior.dscatalog.services.exceptions.DatabaseException;
import com.devsuperior.dscatalog.services.exceptions.ResourceNotFoundException;

@Service
public class ProductService {

	@Autowired
	private ProductRepository repository;
	
	@Autowired
	private CategoryRepository categoryRepository;
	
	@Transactional(readOnly = true)
	public Page<ProductDTO> findAllPaged(Pageable pageable) {
		Page<Product> list = repository.findAll(pageable);
		return list.map(x -> new ProductDTO(x));
	}

	@Transactional(readOnly = true)
	public ProductDTO findById(Long id) {
		Optional<Product> obj = repository.findById(id);
		Product entity = obj.orElseThrow(() -> new ResourceNotFoundException("Entity not found"));
		return new ProductDTO(entity, entity.getCategories());
	}

	@Transactional
	public ProductDTO insert(ProductDTO dto) {
		Product entity = new Product();
		copyDtoToEntity(dto, entity);
		entity = repository.save(entity);
		return new ProductDTO(entity);
	}

	@Transactional
	public ProductDTO update(Long id, ProductDTO dto) {
		try {
			Product entity = repository.getOne(id);
			copyDtoToEntity(dto, entity);
			entity = repository.save(entity);
			return new ProductDTO(entity);
		}
		catch (EntityNotFoundException e) {
			throw new ResourceNotFoundException("Id not found " + id);
		}		
	}

	public void delete(Long id) {
		try {
			repository.deleteById(id);
		}
		catch (EmptyResultDataAccessException e) {
			throw new ResourceNotFoundException("Id not found " + id);
		}
		catch (DataIntegrityViolationException e) {
			throw new DatabaseException("Integrity violation");
		}
	}
	
	private void copyDtoToEntity(ProductDTO dto, Product entity) {

		entity.setName(dto.getName());
		entity.setDescription(dto.getDescription());
		entity.setDate(dto.getDate());
		entity.setImgUrl(dto.getImgUrl());
		entity.setPrice(dto.getPrice());
		
		entity.getCategories().clear();
		for (CategoryDTO catDto : dto.getCategories()) {
			Category category = categoryRepository.getOne(catDto.getId());
			entity.getCategories().add(category);			
		}
	}	
}

```
A classe `ProductService` o método `delete` tem 3 cenários diferentes que podemos utilizar para fazer alguns testes. O teste de unidade não acessa o **repository** que acessa a **BD**. Não temos o raciocínio de quais os dados que vão ser alterados, que resultados devemos esperar, porque o service não tem acesso ao repository real, ou a BD verdadeira na altura de fazermos os testes, logo isso influencia muito na hora de pensar em como posso testar esse service somente olhando para ele.<br/>
Devemos pensar qual a melhor forma de simular o comportamento a nossa dependência, neste caso o `ProductRepository`.

```java
public void delete(Long id) {
	try {
		repository.deleteById(id);
	}
	catch (EmptyResultDataAccessException e) {
		throw new ResourceNotFoundException("Id not found " + id);
	}
	catch (DataIntegrityViolationException e) {
		throw new DatabaseException("Integrity violation");
	}
}
```
#### 1º Teste método `delete(Long id)`
Passamos um **id**, de seguida mandamos apagar esse **id**, e de seguida o que deveria acontecer é que quando o objecto existe, o mesmo será apagado na **BD** e não será lançada nenhuma exceção, e uma vez que o método é **void** não será retornado nada.<br/>
Quando for passado um **id** que existe o método irá chamar o `deleteById` do `repository` e não acontece mais nada do ponto de vista do **service**, no entanto, dentro da **BD** o **id** deverá ser apagado, mas o **service** não vê isso, o **service** faz uma chamada para o `repository` e este toma a iniciativa de apagar os dados na **BD**. Do ponto de vista do **service**, não dará nenhuma exceção e de seguida será chamado o **void**.<br/>
O cenário onde o **id** existe, vai ter que ser chamado o `repository.deleteById(id)` e não acontece mais nada.<br/>

**Configuração:**

```java
@ExtendWith(SpringExtension.class)
public class ProductServiceTests {

	@InjectMocks
	private ProductService service;
	
	@Mock
	private ProductRepository repository;
	
	private Long existingId;
	
	@BeforeEach
	void setUp() throws Exception {
		existingId = 1L;
	}
	
	@Test
	public void deleteShouldDoNothingWhenIdExists() {				
		
		Assertions.assertDoesNotThrow(() -> {
			service.delete(existingId);
		});
	}
}
```


> **Nota:** Se fosse passado um **id** que não existisse cairia na exceção `EmptyResultDataAccessException`e seria chamado o `ResourceNotFoundException`.

Neste momento, apôs a configuração do método `deleteShouldDoNothingWhenIdExists()`, se corrermos o teste na class `ProductServiceTests`, o teste vai terminar sem qualquer erro. Sabemos, no entanto, que o método `delete()` chama o `repository.deleteById(id)`, enquanto no nosso teste, sabemos fazemos o *@InjectMocks* do **service**  e o *@Mock* do **repository**, no entanto, em lugar nenhum configuramos o *@Mock* do `repository` para simular um comportamento do que seria chamar um comportamento do `repository.deleteById(id)` com um **id** que existe. Teremos que configurar o comportamento simulado do *@Mock*. Para simular o comportamento do `deleteById(id)`, então teremos que configurar o que deve acontecer no `deleteById(id)` com um **id** que existe. O teste está incompleto.<br/>
Podemos até fazer uma *Assertion* do *Mockito* para ver se algum método específico do *Mock* foi chamado.

```bash
Mockito.verify(repository).deleteById(existingId);
```
A chamada acima vai verificar se o método `deleteById(existingId)` foi chamado na ação colocada no teste.

```java
Mockito.verify(repository, Mockito.times(1)).deleteById(existingId);
```
No código acima, estamos a testar se o delete do `repository` foi chamado apenas uma vez.

```java
Mockito.verify(repository, Mockito.never()).deleteById(existingId);
```
No código acima estamos a testar se o delete do `repository` não foi testado nenhuma vez. 
De qualquer forma, temos que configurar o comportamento simulado do método `deleteById(existingId)`, e ainda não foi feito.
```java
@Test
public void deleteShouldDoNothingWhenIdExists() {				
	
	Assertions.assertDoesNotThrow(() -> {
		service.delete(existingId);
	});
	
	Mockito.verify(repository).deleteById(existingId);
}
```
Podemos configurar um comportamento simulado para o caso de ser passado um **id** que não existe na **BD**: <br/>

Criamos um **id** privado
```bash
private Long nonExistingId;
```
Configuramos o **id** que não existe no `service` 
```bash
nonExistingId = 1000L;
```
Fazemos a configuração no **Mockito** no nosso `repository`para esse valor
```bash
Mockito.doThrow(EmptyResultDataAccessException.class).when(repository).deleteById(nonExistingId);
```

```bash
@BeforeEach
void setUp() throws Exception {
	(...)
	nonExistingId = 1000L;		
	(...)		
	Mockito.doThrow(EmptyResultDataAccessException.class).when(repository).deleteById(nonExistingId);
}
```
Basicamente, configuramos os comportamentos para o nosso objecto **Mockado**.<br/>
Agora se chamarmos o `service`passando o **id** existente o JUnity irá chamar o **id** que configuramos no **Mockito**, e nesse caso, não faz nada,e o teste passa com sucesso.

```java
@Test
public void deleteShouldDoNothingWhenIdExists() {				
	
	Assertions.assertDoesNotThrow(() -> {
		service.delete(nonExistingId);
	});
	
	Mockito.verify(repository, Mockito.times(1)).deleteById(nonExistingId);
}
```
No exemplo em cima testamos a implementação para um **id** que não existe. Neste caso, o teste tem que dar erro (vermelho), uma vez que o **id** 1000 não existe. A exceção foi lançada, pelo comportamento simulado no **Mockito**.

##  Imports estáticos do Mockito
>É possível utilizarmos os métodos do **Mockito** sem fazermos a referência `Mockito.doNothing()` por exemplo. A utilização da classe não é obrigatória, no entanto é mais didático:

```java
import static org.mockito.Mockito.doNothing;
import static org.mockito.Mockito.doThrow;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;

import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.springframework.dao.EmptyResultDataAccessException;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import com.devsuperior.dscatalog.repositories.ProductRepository;

@ExtendWith(SpringExtension.class)
public class ProductServiceTests {

	@InjectMocks
	private ProductService service;
	
	@Mock
	private ProductRepository repository;
	
	private Long existingId;
	private Long nonExistingId;
	
	@BeforeEach
	void setUp() throws Exception {
		existingId = 1L;
		nonExistingId = 1000L;
		
		doNothing().when(repository).deleteById(existingId);
		
		doThrow(EmptyResultDataAccessException.class).when(repository).deleteById(nonExistingId);
	}
	
	@Test
	public void deleteShouldDoNothingWhenIdExists() {				
		
		Assertions.assertDoesNotThrow(() -> {
			service.delete(existingId);
		});
		
		verify(repository, times(1)).deleteById(existingId);
	}
}
```
## Teste delete lança ResourceNotFoundException quando id não existe
Neste 2º cenário, o utilizador informa um **id** que não existe e a classe `ProductService` deveria lançar a exceção `ResourceNotFoundException`. Aqui,quando o **id** não existir deverá ser lançada a exceção.<br />
O código final irá ficar com a chamada para o **id** que não existe:

```java
@Test
	public void deleteShouldThrowResourceNotFoundExceptionWhenIdDoesNotExist() {				
		
		Assertions.assertThrows(ResourceNotFoundException.class, () -> {
			service.delete(nonExistingId);
		});
		
		Mockito.verify(repository, Mockito.times(1)).deleteById(nonExistingId);
	}
```
Por último, devemos verificar no **Mockito** se foi chamado o `nonExistingID`.

## Teste delete lança DatabaseException quando id dependente
No processo de ralizção de testes do método `delete()`, vamos testar o processo em que é violada a integredidade da **BD**. Poderá ser por exemplo, uma tentativa de apagar um objecto que estará associado com outro, e teria nesse caso que lançada a exceção `DatabaseException`, e o **repository** iria lançar o `DataIntegrityViolationException`, logo, iremos simular primeiramente esse comportamento no **repository**. Vamos simular o comportamento de apagar um objecto que está associado com outro e por questões de integridade referenciais, por exemplo ao apagar um deles a chave estrangeira do outro fica sem dono, essa é uma situação que não devemos permitir que aconteça. Neste caso, devemos lançar a exceção `DataIntegrityViolationException` no nosso **repository**.

Comportamento simulado da situação reportada:

```bach
private long dependentId;
```
```bash
dependentId = 4L;
```
```bach
Mockito.doThrow(DataIntegrityViolationException.class).when(repository).deleteById(dependentId);
```
```java
@ExtendWith(SpringExtension.class)
public class ProductServiceTests {

	@InjectMocks
	private ProductService service;
	
	@Mock
	private ProductRepository repository;
	
	private long existingId;
	private long nonExistingId;
	private long dependentId;
	
	@BeforeEach
	void setUp() throws Exception {
		existingId = 1L;
		nonExistingId = 1000L;
		dependentId = 4L;
		
		Mockito.doNothing().when(repository).deleteById(existingId);
		
		Mockito.doThrow(EmptyResultDataAccessException.class).when(repository).deleteById(nonExistingId);

		Mockito.doThrow(DataIntegrityViolationException.class).when(repository).deleteById(dependentId);
	}
```
> Neste caso, no atual sistema que montamos, não existiria problemas ao apagar um produto. Os produtos são entidades independentes. A entidade das categorias é que não podem ser independentes. Podemos apagar produtos sem problema algum, não poderíamos é apagar categorias com produtos associados.<br />
No atual sistema montado fica já preparado para que num futuro, caso seja implementada uma classe de pedidos associados a um produto, essa validação já estaria feita. Não seria nesse caso, possível apagar um produto com pedidos associados a ele.

## Simulando comportamentos diversos com Mockito

Simulação para outros métodos do `repository`, classe **ProductRepository**, simulando o comportamento do **ProductRepository**.

### Simulação do comportamento do método `findAll(pageable)`

O `findAll(pageable)` retorna uma um objecto do tipo `Page<Product>`. O objecto *Mockado* tem que retornar também um *pageable*

```java
Page<Product> list = repository.findAll(pageable);
```
Nexste caso, o método que vamos simular retorna um valor `Page<Product>`

```bash
import org.springframework.data.domain.PageImpl;
```
O `PageImpl` representa um tipo concreto que representa uma página de dados para nós, e é muito usado para testes.

```java
import java.util.List;

(...)
import org.springframework.data.domain.PageImpl;
import org.springframework.data.domain.Pageable;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import com.devsuperior.dscatalog.entities.Product;
import com.devsuperior.dscatalog.repositories.ProductRepository;
(...)
import com.devsuperior.dscatalog.tests.Factory;

@ExtendWith(SpringExtension.class)
public class ProductServiceTests {

	@InjectMocks
	private ProductService service;
	
	@Mock
	private ProductRepository repository;
	
	private long existingId;
	private long nonExistingId;
	private long dependentId;
	private PageImpl<Product> page;
	private Product product;
	
	@BeforeEach
	void setUp() throws Exception {
		existingId = 1L;
		nonExistingId = 2L;
		dependentId = 3L;
		product = Factory.createProduct();
		page = new PageImpl<>(List.of(product));
		
		Mockito.when(repository.findAll((Pageable)ArgumentMatchers.any())).thenReturn(page);
		
		Mockito.doNothing().when(repository).deleteById(existingId);		
		Mockito.doThrow(EmptyResultDataAccessException.class).when(repository).deleteById(nonExistingId);
		Mockito.doThrow(DataIntegrityViolationException.class).when(repository).deleteById(dependentId);
	}
```
### Simulação do comportamento do método `save(entity)`
O método `save(entity)` recebe uma **entity** como argumento e retorna uma referência para a **entity**.

Classe `ProductService`:

```java
(...)
@Service
public class ProductService {

	@Autowired
	private ProductRepository repository;
	
	@Autowired
	private CategoryRepository categoryRepository;
	
	@Transactional(readOnly = true)
	public Page<ProductDTO> findAllPaged(Pageable pageable) {
		Page<Product> list = repository.findAll(pageable);
		return list.map(x -> new ProductDTO(x));
	}

	@Transactional(readOnly = true)
	public ProductDTO findById(Long id) {
		Optional<Product> obj = repository.findById(id);
		Product entity = obj.orElseThrow(() -> new ResourceNotFoundException("Entity not found"));
		return new ProductDTO(entity, entity.getCategories());
	}

	@Transactional
	public ProductDTO insert(ProductDTO dto) {
		Product entity = new Product();
		copyDtoToEntity(dto, entity);
		entity = repository.save(entity);
		return new ProductDTO(entity);
	}

	@Transactional
	public ProductDTO update(Long id, ProductDTO dto) {
		try {
			Product entity = repository.getOne(id);
			copyDtoToEntity(dto, entity);
			entity = repository.save(entity);
			return new ProductDTO(entity);
		}
		catch (EntityNotFoundException e) {
			throw new ResourceNotFoundException("Id not found " + id);
		}		
	}
	(...)
```

No caso em baixo, quando for chamado o `save(entity)` deverá ser retornado um `ProductDTO(entity)`:

```bash
@Transactional
public ProductDTO insert(ProductDTO dto) {
	Product entity = new Product();
	copyDtoToEntity(dto, entity);
	entity = repository.save(entity);
	return new ProductDTO(entity);
}
```
```bash
@Transactional
public ProductDTO update(Long id, ProductDTO dto) {
	try {
		Product entity = repository.getOne(id);
		copyDtoToEntity(dto, entity);
		entity = repository.save(entity);
		return new ProductDTO(entity);
	}
	catch (EntityNotFoundException e) {
		throw new ResourceNotFoundException("Id not found " + id);
	}		
}
```
Implementação do método simulado na classe `ProductServiceTests` simulando o comportamento do método `save(entity)` do **repository**:

```bash
Mockito.when(repository.save(ArgumentMatchers.any())).thenReturn(product);
```
```java
@BeforeEach
void setUp() throws Exception {
	existingId = 1L;
	nonExistingId = 2L;
	dependentId = 3L;
	product = Factory.createProduct();
	page = new PageImpl<>(List.of(product));
	
	Mockito.when(repository.findAll((Pageable)ArgumentMatchers.any())).thenReturn(page);
	
	Mockito.when(repository.save(ArgumentMatchers.any())).thenReturn(product);
	
	Mockito.doNothing().when(repository).deleteById(existingId);		
	Mockito.doThrow(EmptyResultDataAccessException.class).when(repository).deleteById(nonExistingId);
	Mockito.doThrow(DataIntegrityViolationException.class).when(repository).deleteById(dependentId);
}
```
Exemplo de implementação do método `findById(existingId)` e `findById(nonExistingId)` na classe `ProductServiceTests`:

```bash
Mockito.when(repository.findById(existingId)).thenReturn(Optional.of(product));
Mockito.when(repository.findById(nonExistingId)).thenReturn(Optional.empty());
```
Código final:

```java
(...)
public class ProductServiceTests {

	@InjectMocks
	private ProductService service;
	
	@Mock
	private ProductRepository repository;
	
	private long existingId;
	private long nonExistingId;
	private long dependentId;
	private PageImpl<Product> page;
	private Product product;
	
	@BeforeEach
	void setUp() throws Exception {
		existingId = 1L;
		nonExistingId = 2L;
		dependentId = 3L;
		product = Factory.createProduct();
		page = new PageImpl<>(List.of(product));
		
		Mockito.when(repository.findAll((Pageable)ArgumentMatchers.any())).thenReturn(page);
		
		Mockito.when(repository.save(ArgumentMatchers.any())).thenReturn(product);
		
		Mockito.when(repository.findById(existingId)).thenReturn(Optional.of(product));
		Mockito.when(repository.findById(nonExistingId)).thenReturn(Optional.empty());
		
		Mockito.doNothing().when(repository).deleteById(existingId);		
		Mockito.doThrow(EmptyResultDataAccessException.class).when(repository).deleteById(nonExistingId);
		Mockito.doThrow(DataIntegrityViolationException.class).when(repository).deleteById(dependentId);
	}
(...)
```
## Testando findAllPaged do ProductService

```java
@Test
public void findAllPagedShouldReturnPaged() {
	
	Pageable pageable = PageRequest.of(0, 10);
	
	Page<ProductDTO> result = service.findAllPaged(pageable);
	
	Assertions.assertNotNull(result);
	Mockito.verify(repository, Mockito.times(1)).findAll(pageable);
}
```

## Exercícios: testes de unidade com Mockito

*Solução:* https://youtu.be/KvXL5HgX5Jg

Favor implementar os seguintes testes em **`ProductServiceTests`**:

-	**findById** deveria 
	-	retornar um `ProductDTO` quando o id existir
	-	lançar `ResourceNotFoundException` quando o id não existir
-	**update** deveria (dica: você vai ter que simular o comportamento do getOne)
	-	retornar um `ProductDTO` quando o id existir
	-	lançar uma `ResourceNotFoundException` quando o id não existir

```java
@Test
public void findByIdShouldThrowResourceNotFoundExceptionWhenIdDoesNotExist() {
	
	Assertions.assertThrows(ResourceNotFoundException.class, () -> {
		service.findById(nonExistingId);
	});
	
}

@Test
public void findByIdShouldReturnProductDTOWhenIdExist() {
	ProductDTO result = service.findById(existingId);
	Assertions.assertNotNull(result);
}
```

Mock do ProductRepository:

```java
@Mock
private ProductRepository repository;
```

Já temos o Mock do Product Repository falta fazer a simulação do comportamento do getOne()

```java
Mockito.when(repository.getOne(existingId)).thenReturn(product);
Mockito.when(repository.getOne(nonExistingId)).thenThrow(EntityNotFoundException.class);
```
Nota que quando o getOne chama um nonExistingId, é lançada a exceção EntityNotFoundException, que não é a nossa exceção de serviço que é da JPA.

```bash
import javax.persistence.EntityNotFoundException;
```
Simulação do `CategoryRepository`

```bash
@Mock
private CategoryRepository categoryRepository;
```
Se for chamado o categoryRepository num existingId vamos ter que criar uma nova categoria, logo devemos instanciar uma nova categoria:

```bash
private Category category;
```
Na classe ``Factory`, iremos criar um método para criar categorias:

```bash
public static Category createCategory() {
	Category category = new Category(1L, "Computers");
	return category;
}
```
Na class `ProductServiceTests`configuramos a variavel **category** para receber o Factory da criação da categoria:

```bash
@BeforeEach
void setUp() throws Exception {
	(...)
	category = Factory.createCategory();
```
Agora já podemos usar no Mockito o category para simular o comportamenta `CategoryRepository`:

```java
Mockito.when(categoryRepository.getOne(existingId)).thenReturn(category);
Mockito.when(categoryRepository.getOne(nonExistingId)).thenThrow(EntityNotFoundException.class);
```

Teste ao método `update()` e retornar um `ProductDTO` quando o **id** existir:

```bash
@Test
public void updateShouldThrowResourceNotFoundExceptionWhenIdDoesNotExist() {
	
	Assertions.assertThrows(ResourceNotFoundException.class, () -> {
		service.update(nonExistingId, productDTO);
	});
	
}
```
```java
(...)
private ProductDTO productDTO;
	
@BeforeEach
void setUp() throws Exception {
	existingId = 1L;
	nonExistingId = 2L;
	dependentId = 3L;
	product = Factory.createProduct();
	page = new PageImpl<>(List.of(product));
	category = Factory.createCategory();
	productDTO = Factory.createProductDTO();
			
	Mockito.when(repository.findAll((Pageable)ArgumentMatchers.any())).thenReturn(page);
	
	Mockito.when(repository.save(ArgumentMatchers.any())).thenReturn(product);
	
	Mockito.when(repository.findById(existingId)).thenReturn(Optional.of(product));
	Mockito.when(repository.findById(nonExistingId)).thenReturn(Optional.empty());
	
	Mockito.when(repository.getOne(existingId)).thenReturn(product);
	Mockito.when(repository.getOne(nonExistingId)).thenThrow(EntityNotFoundException.class);
	
	Mockito.when(categoryRepository.getOne(existingId)).thenReturn(category);
	Mockito.when(categoryRepository.getOne(nonExistingId)).thenThrow(EntityNotFoundException.class);
	
	Mockito.doNothing().when(repository).deleteById(existingId);		
	Mockito.doThrow(EmptyResultDataAccessException.class).when(repository).deleteById(nonExistingId);
	Mockito.doThrow(DataIntegrityViolationException.class).when(repository).deleteById(dependentId);
}

@Test
public void updateShouldThrowResourceNotFoundExceptionWhenIdDoesNotExist() {
	
	Assertions.assertThrows(ResourceNotFoundException.class, () -> {
		service.update(nonExistingId, productDTO);
	});
	
}
(...)
```

## Começando testes na camada web
Iniciando testes com os controladores ou resources, no caso vamos trabalhar com o `ProductResource`.
Pasta **src/test/java** package **com.devsuperior.dscatalog** e a partir daqui criamos uma nova classe.
A classe original está no package **com.devsuperior.dscatalog.resources;** logo, na criação da nossa classe devemos cria-la dentro desse mesmo package (com.devsuperior.dscatalog.resources). Classe **ProductResourceTests**

![product_resource_tests](https://user-images.githubusercontent.com/22635013/137274679-b98e3209-9a9b-4199-81df-a20aa7cf5ca9.JPG)
É criado o novvo package:
![product_resource_tests_package](https://user-images.githubusercontent.com/22635013/137275082-a64fca2b-c218-4322-9e7d-1ca808472f9b.JPG)

A criação da classe obriga a fazermos ou utilizarmos uma Annotations. No caso, temos duas opções básicas:
- @SpringBootTest e @AutoConfigureMockMvc - Carrega o contexto da aplicação (teste de integração & web)
Trata as requisições sem subir o servidor

- @WebMvcTest(Classe.class) - Carrega o contexto, porém somente da camada web (teste de unidade: controlador)

Utilizamos a 2ª opção

```java
package com.devsuperior.dscatalog.resources;

import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;

@WebMvcTest(ProductResourceTests.class)
public class ProductResourceTests {

}

```
Para fazermos testes na camada web, vamos fazer requesições, logo iremos chamar o endpoint. Para chamar o endpoint nos testes, uma abordagem bastante utilizada é o MockMvc em conjunto com a Annotation *@Autowired* para injectar o tipo de componente:

```java
package com.devsuperior.dscatalog.resources;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.web.servlet.MockMvc;

@WebMvcTest(ProductResourceTests.class)
public class ProductResourceTests {

	@Autowired
	private MockMvc mockMvc;
}
```
Como o `ProductResources`tem uma dependência do `ProductService` também devemos declarar essa dependência no `ProductResourceTests`

![product_resource_ProductService_dependence](https://user-images.githubusercontent.com/22635013/137278496-1862d73d-8c02-4928-b541-17c21eabbc9b.JPG)
*ProductResource*

No entanto, a dependência `ProductService` não é para ser integrada com o service, teremos que *Mockar* o componente, como tal usamos a Annotation @MockBean porque devemos dar preferência a esta Annotation sempre que usamos testes de @WebMvcTest ou @SpringBootTest 

### Testar o método `findAll()` do `ProductResource`

```bash
@GetMapping
public ResponseEntity<Page<ProductDTO>> findAll(Pageable pageable) {		
	// PARÂMETROS: page, size, sort						
	Page<ProductDTO> list = service.findAllPaged(pageable);		
	return ResponseEntity.ok().body(list);
}
```
*Class ProductResource*
Como vemos este método retorna uma página de produtos DTO - **ProductDTO**.

Vamos iniciar um teste de unidade simulando o comportamento do *service*, e esse teste tem que expecificar que está a retornar um objecto do tipo página - **Page\<ProductDTO>**.

1º  Simular o comportamento do service porque o findAll() do chama o findAllPaged do service - **service.findAllPaged(pageable)** 5:42



Declaramos o método setUp() para declarar algumas variáveis e setar alguns comportamentos que vamos ter que definir, do ProductService.

Temos de simular o comportamento do findAllPaged do service - **service.findAllPaged(pageable)**

```bash
@BeforeEach
void setUp() throws Exception {
	
	productDTO = Factory.createProductDTO();
	page = new PageImpl<>(List.of(productDTO));
	
	when(service.findAllPaged(ArgumentMatchers.any())).thenReturn(page);
}
```
O **PageImpl** é um tipo concreto de página, de implementação. Serve para instanciar um objecto concreto.

```bash
@Test
public void findAllShouldReturnPage() throws Exception {
	mockMvc.perform(get("/products")).andExpect(status().isOk());
}
```

O `mockMvc`faz uma requisição com o método http `get()`e nele temos que colocar o caminho **products**

> Depois de testar o STS devolveu o erro `java.lang.AssertionError: Status expected:<200> but was:<404>
Expected :200
Actual   :404`

Apôs alterar para a Notation abaixo o erro desapareceu:
```java
@SpringBootTest
@AutoConfigureMockMvc
public class ProductResourceTests {

	@Autowired
	private MockMvc mockMvc;

```
## Testando o `findById()`

Para continuar com os nossos testes, vamos prosseguir com o `findById()`. Temos que simular primeiro o comportamento do `findById()` da classe **ProductService**

`findById` class **ProductResource**:

```bash
@GetMapping(value = "/{id}")
public ResponseEntity<ProductDTO> findById(@PathVariable Long id) {
	ProductDTO dto = service.findById(id);
	return ResponseEntity.ok().body(dto);
}
```

O `findByd` da classe **ProductService** ou faz o retorno do **ProductDTO** ou lança a exceção **ResourceNotFoundException**, logo vamos ter que simular o comportamento do **ProductService**:

```bash
@Transactional(readOnly = true)
public ProductDTO findById(Long id) {
	Optional<Product> obj = repository.findById(id);
	Product entity = obj.orElseThrow(() -> new ResourceNotFoundException("Entity not found"));
	return new ProductDTO(entity, entity.getCategories());
}
```

Implementação do `findById()` do service no **ProductResourceTests**:

```java
(...)
@MockBean
private ProductService service;

private Long existingId;
private Long nonExistingId;
private ProductDTO productDTO;
private PageImpl<ProductDTO> page;

@BeforeEach
void setUp() throws Exception {
	
	existingId = 1L;
	nonExistingId = 2L;
	
	productDTO = Factory.createProductDTO();		
	page = new PageImpl<>(List.of(productDTO));
	
	when(service.findAllPaged(any())).thenReturn(page);
	
	when(service.findById(existingId)).thenReturn(productDTO);
	when(service.findById(nonExistingId)).thenThrow(ResourceNotFoundException.class);
}
(...)
```
Implementação do `findById()` do resource na camada web no **ProductResourceTests**:

```java
(...)

@SpringBootTest
@AutoConfigureMockMvc
public class ProductResourceTests {

	@Autowired
	private MockMvc mockMvc;
	
	@MockBean
	private ProductService service;
	
	private Long existingId;
	private Long nonExistingId;
	private ProductDTO productDTO;
	private PageImpl<ProductDTO> page;
	
	@BeforeEach
	void setUp() throws Exception {
		
		existingId = 1L;
		nonExistingId = 2L;
		
		productDTO = Factory.createProductDTO();		
		page = new PageImpl<>(List.of(productDTO));
		
		when(service.findAllPaged(any())).thenReturn(page);
		
		when(service.findById(existingId)).thenReturn(productDTO);
		when(service.findById(nonExistingId)).thenThrow(ResourceNotFoundException.class);
	}
	
	@Test
	public void findAllShouldReturnPage() throws Exception  {
		/*
		mockMvc.perform(get("/products")).andExpect(status().isOk());
		*/		
		ResultActions result = 
				mockMvc.perform(get("/products")
						.accept(MediaType.APPLICATION_JSON));
		
		result.andExpect(status().isOk());
		
	}
	
	@Test
	public void findByIdShouldReturnProductWhenIdExists() throws Exception {
		/*
		mockMvc.perform(get("/products/{id}", existingId)).andExpect(status().isOk());
		*/
		ResultActions result = 
				mockMvc.perform(get("/products/{id}", existingId)
						.accept(MediaType.APPLICATION_JSON));
		
		result.andExpect(status().isOk());
		result.andExpect(jsonPath("$.id").exists());
		result.andExpect(jsonPath("$.name").exists());
		result.andExpect(jsonPath("$.description").exists());
		
	}
	
	@Test
	public void findByIdShouldReturnNotFoundExcetionWhenIdDoesNotExist() throws Exception {
		
		ResultActions result = 
				mockMvc.perform(get("/products/{id}", nonExistingId)
						.accept(MediaType.APPLICATION_JSON));
		
		result.andExpect(status().isNotFound());
	}
}
```
### Testando o método `update`
Teste simulado na camada web com comportamento simulado das dependências com o método `update` da classe **ProductResource** 

```java
@PutMapping(value = "/{id}")
public ResponseEntity<ProductDTO> update(@PathVariable Long id, @RequestBody ProductDTO dto) {
	dto = service.update(id, dto);
	return ResponseEntity.ok().body(dto);
}
```

O `update` recebe um **@PathVariable Long id** e um  **@RequestBody ProductDTO dto**, faz também uma chamada ao `update` do **service** e atualiza o objeto retornando um objecto **ProductDTO**

O **service** pode guardar os dados ou lançar umaexceçao, veja-se na classe **ProductService**:

```java
@Transactional
public ProductDTO update(Long id, ProductDTO dto) {
	try {
		Product entity = repository.getOne(id);
		copyDtoToEntity(dto, entity);
		entity = repository.save(entity);
		return new ProductDTO(entity);
	}
	catch (EntityNotFoundException e) {
		throw new ResourceNotFoundException("Id not found " + id);
	}		
}
```
Simular o comportamento do **service**, é parecido com o método `findById`, recebe no entanto, mais um parâmetro que é o **ProductDTO**, no entanto utilizamos o `any()` para simular o comportamento de qualquer objecto retornado.

```bash
@BeforeEach
void setUp() throws Exception {
	
	existingId = 1L;
	nonExistingId = 2L;
	
	productDTO = Factory.createProductDTO();		
	page = new PageImpl<>(List.of(productDTO));
	
	when(service.findAllPaged(any())).thenReturn(page);
	
	when(service.findById(existingId)).thenReturn(productDTO);
	when(service.findById(nonExistingId)).thenThrow(ResourceNotFoundException.class);

	when(service.update(existingId, any())).thenReturn(productDTO);
	when(service.update(nonExistingId, any())).thenThrow(ResourceNotFoundException.class);
}
```

```java
@Test
public void updateShouldReturnProductDTOWhenIdExists() throws Exception {
	
	String jsonBody = objectMapper.writeValueAsString(productDTO);
	
	ResultActions result = 
			mockMvc.perform(put("/products/{id}", existingId)
					.content(jsonBody)
					.contentType(MediaType.APPLICATION_JSON)
					.accept(MediaType.APPLICATION_JSON));
	
	result.andExpect(status().isOk());
	result.andExpect(jsonPath("$.id").exists());
	result.andExpect(jsonPath("$.name").exists());
	result.andExpect(jsonPath("$.description").exists());
}
```

> O `put` é uma requisição que tem corpo, ou seja, temos que passar um objeto completo com os seus dados (e.g.: id, name, description), tal como podemos ver no *Postman*, só que esse objeto não é java mas sim um objeto *Json*, logo temos que converter esse objeto de java para *Json* e isso é feito com o **ObjectMapper**.
O **ProductDTO** deverá ser assim transformado em *String* para o *Json*:

```bash
String jsonBody = objectMapper.writeValueAsString(productDTO);
```

Passar o String jsonBody na requisição `.content(jsonBody)`. 
Precisamos de negociar também o tipo de datos da requisição, não só o da resposta, que é o `accept`, isso é feito com o `.contentType(MediaType.APPLICATION_JSON)`. 

De seguida basta fazermos as nossas Assertions:

```bash
result.andExpect(status().isOk());
result.andExpect(jsonPath("$.id").exists());
result.andExpect(jsonPath("$.name").exists());
result.andExpect(jsonPath("$.description").exists());
```

Se corrermos o teste, vai falhar em todos os testes por causa do ArgumentMatchers. Quando utilizamos o argumento `any()` os outros argumentos não podem ser argumentos simples:

![argument_matchers](https://user-images.githubusercontent.com/22635013/138419127-13ee8d45-4a75-4f35-b612-7b21eafb9aac.JPG)

Temos que usar o método `eq()`: 
```java
when(service.update(eq(existingId), any())).thenReturn(productDTO);
when(service.update(eq(nonExistingId), any())).thenThrow(ResourceNotFoundException.class);
```
![argument_eq](https://user-images.githubusercontent.com/22635013/138419969-a63e7725-e8e4-4c11-b7eb-9d37bedd2a7a.JPG)

Aspecto da classe **ProductResourceTests** no final da implementação do `update()`:

```java
package com.devsuperior.dscatalog.resources;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.put;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import java.util.List;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.data.domain.PageImpl;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.ResultActions;

import com.devsuperior.dscatalog.dto.ProductDTO;
import com.devsuperior.dscatalog.services.ProductService;
import com.devsuperior.dscatalog.services.exceptions.ResourceNotFoundException;
import com.devsuperior.dscatalog.tests.Factory;
import com.fasterxml.jackson.databind.ObjectMapper;

@SpringBootTest
@AutoConfigureMockMvc
public class ProductResourceTests {

	@Autowired
	private MockMvc mockMvc;
	
	@MockBean
	private ProductService service;
	
	@Autowired
	private ObjectMapper objectMapper;
	
	private Long existingId;
	private Long nonExistingId;
	private ProductDTO productDTO;
	private PageImpl<ProductDTO> page;
	
	@BeforeEach
	void setUp() throws Exception {
		
		existingId = 1L;
		nonExistingId = 2L;
		
		productDTO = Factory.createProductDTO();		
		page = new PageImpl<>(List.of(productDTO));
		
		when(service.findAllPaged(any())).thenReturn(page);
		
		when(service.findById(existingId)).thenReturn(productDTO);
		when(service.findById(nonExistingId)).thenThrow(ResourceNotFoundException.class);

		when(service.update(eq(existingId), any())).thenReturn(productDTO);
		when(service.update(eq(nonExistingId), any())).thenThrow(ResourceNotFoundException.class);
	}
	
	@Test
	public void updateShouldReturnProductDTOWhenIdExists() throws Exception {
		
		String jsonBody = objectMapper.writeValueAsString(productDTO);
		
		ResultActions result = 
				mockMvc.perform(put("/products/{id}", existingId)
						.content(jsonBody)
						.contentType(MediaType.APPLICATION_JSON)
						.accept(MediaType.APPLICATION_JSON));
		
		result.andExpect(status().isOk());
		result.andExpect(jsonPath("$.id").exists());
		result.andExpect(jsonPath("$.name").exists());
		result.andExpect(jsonPath("$.description").exists());
	}

	@Test
	public void updateShouldReturnNotFoundWhenIdDoesNotExists() throws Exception {
		
	}
	
	@Test
	public void findAllShouldReturnPage() throws Exception  {
		/*
		mockMvc.perform(get("/products")).andExpect(status().isOk());
		*/		
		ResultActions result = 
				mockMvc.perform(get("/products")
						.accept(MediaType.APPLICATION_JSON));
		
		result.andExpect(status().isOk());
		
	}
	
	@Test
	public void findByIdShouldReturnProductWhenIdExists() throws Exception {
		/*
		mockMvc.perform(get("/products/{id}", existingId)).andExpect(status().isOk());
		*/
		ResultActions result = 
				mockMvc.perform(get("/products/{id}", existingId)
						.accept(MediaType.APPLICATION_JSON));
		
		result.andExpect(status().isOk());
		result.andExpect(jsonPath("$.id").exists());
		result.andExpect(jsonPath("$.name").exists());
		result.andExpect(jsonPath("$.description").exists());
		
	}
	
	@Test
	public void findByIdShouldReturnNotFoundExcetionWhenIdDoesNotExist() throws Exception {
		
		ResultActions result = 
				mockMvc.perform(get("/products/{id}", nonExistingId)
						.accept(MediaType.APPLICATION_JSON));
		
		result.andExpect(status().isNotFound());
	}
}
```
### Simulando outros comportamentos com do **ProductService**

```java
public void delete(Long id) {
	try {
		repository.deleteById(id);
	}
	catch (EmptyResultDataAccessException e) {
		throw new ResourceNotFoundException("Id not found " + id);
	}
	catch (DataIntegrityViolationException e) {
		throw new DatabaseException("Integrity violation");
	}
}
```

Comportamentos simulados do método `delete`, que é um método void. O método apresenta 3 cenários, no qual apaga e não faz mais nada quando o **id** existe, quando **id** não existe ele lança a exceção `ResourceNotFoundException`, e por último quando o objecto tem uma associação com outro objecto lança a exceção `DatabaseException`.

Iremos fazer o método simulado no *mock*, lembrando que quando o método é void iniciamos primeiro a consequência e só depois o when.

Simulando o **id** existente:

```bash
doNothing().when(service).delete(existingId);
```
Quando for chamado o **service** com o método `delete(existinghId)`, ou seja o **id** existente, não faz nada.

Simulando o **id** não existente, neste caso teremos que retornar o `ResourceNotFoundException`:

```bash
doThrow(ResourceNotFoundException.class).when(service).delete(nonExistingId);
```

Simulando o mesmo comportamento para o **id** que tem o objecto associado e que viola a integridade:

```bash
doThrow(DatabaseException.class).when(service).delete(dependentId);
```
Aspecto da classe **ProductResourceTests** no final da implementação do `delete()`:

```java
package com.devsuperior.dscatalog.resources;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.doNothing;
import static org.mockito.Mockito.doThrow;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.put;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import java.util.List;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.data.domain.PageImpl;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.ResultActions;

import com.devsuperior.dscatalog.dto.ProductDTO;
import com.devsuperior.dscatalog.services.ProductService;
import com.devsuperior.dscatalog.services.exceptions.DatabaseException;
import com.devsuperior.dscatalog.services.exceptions.ResourceNotFoundException;
import com.devsuperior.dscatalog.tests.Factory;
import com.fasterxml.jackson.databind.ObjectMapper;

@SpringBootTest
@AutoConfigureMockMvc
public class ProductResourceTests {

	@Autowired
	private MockMvc mockMvc;
	
	@MockBean
	private ProductService service;
	
	@Autowired
	private ObjectMapper objectMapper;
	
	private Long existingId;
	private Long nonExistingId;
	private Long dependentId;
	private ProductDTO productDTO;
	private PageImpl<ProductDTO> page;
	
	@BeforeEach
	void setUp() throws Exception {
		
		existingId = 1L;
		nonExistingId = 2L;
		dependentId = 3L;
		
		productDTO = Factory.createProductDTO();		
		page = new PageImpl<>(List.of(productDTO));
		
		when(service.findAllPaged(any())).thenReturn(page);
		
		when(service.findById(existingId)).thenReturn(productDTO);
		when(service.findById(nonExistingId)).thenThrow(ResourceNotFoundException.class);

		when(service.update(eq(existingId), any())).thenReturn(productDTO);
		when(service.update(eq(nonExistingId), any())).thenThrow(ResourceNotFoundException.class);
		
		doNothing().when(service).delete(existingId);
		doThrow(ResourceNotFoundException.class).when(service).delete(nonExistingId);
		doThrow(DatabaseException.class).when(service).delete(dependentId);
		
		
	}
	
	@Test
	public void updateShouldReturnProductDTOWhenIdExists() throws Exception {
		
		String jsonBody = objectMapper.writeValueAsString(productDTO);
		
		ResultActions result = 
				mockMvc.perform(put("/products/{id}", existingId)
						.content(jsonBody)
						.contentType(MediaType.APPLICATION_JSON)
						.accept(MediaType.APPLICATION_JSON));
		
		result.andExpect(status().isOk());
		result.andExpect(jsonPath("$.id").exists());
		result.andExpect(jsonPath("$.name").exists());
		result.andExpect(jsonPath("$.description").exists());
	}

	@Test
	public void updateShouldReturnNotFoundWhenIdDoesNotExists() throws Exception {
		
	}
	
	@Test
	public void findAllShouldReturnPage() throws Exception  {
		/*
		mockMvc.perform(get("/products")).andExpect(status().isOk());
		*/		
		ResultActions result = 
				mockMvc.perform(get("/products")
						.accept(MediaType.APPLICATION_JSON));
		
		result.andExpect(status().isOk());
		
	}
	
	@Test
	public void findByIdShouldReturnProductWhenIdExists() throws Exception {
		/*
		mockMvc.perform(get("/products/{id}", existingId)).andExpect(status().isOk());
		*/
		ResultActions result = 
				mockMvc.perform(get("/products/{id}", existingId)
						.accept(MediaType.APPLICATION_JSON));
		
		result.andExpect(status().isOk());
		result.andExpect(jsonPath("$.id").exists());
		result.andExpect(jsonPath("$.name").exists());
		result.andExpect(jsonPath("$.description").exists());
		
	}
	
	@Test
	public void findByIdShouldReturnNotFoundExcetionWhenIdDoesNotExist() throws Exception {
		
		ResultActions result = 
				mockMvc.perform(get("/products/{id}", nonExistingId)
						.accept(MediaType.APPLICATION_JSON));
		
		result.andExpect(status().isNotFound());
	}
}
```



## Autor
Lenine Ferrer de Pestana <br />
Email: leninepestana@gmail.com



