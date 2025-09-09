Apresentação das pastas
src/
├── main/
│   └── java/
│       └── io/
│           └── company/
│               ├── authors/
│               │   ├── application/
│               │   │   ├── commands/
│               │   │   │   ├── UpdateAuthorCommand.java
│               │   │   │   └── ...commands
│               │   │   ├── handlers/
│               │   │   │   ├── UpdateAuthorHandler.java
│               │   │   │   └── ...handlers
│               │   │   ├── strategies/
│               │   │   │   ├── AuthorCreationStrategy.java
│               │   │   │   ├── AuthorUpdateStrategy.java
│               │   │   │   ├── IndieAuthorStrategy.java
│               │   │   │   ├── AuthorCreationStrategy.java
│               │   │   │   ├── AuthorStrategyFactory.java
│               │   │   └── events/
│               │   │       ├── AuthorUpdatedEvent.java
│               │   │       └── ...events
│               │   ├── domain/
│               │   │   ├── Author.java
│               │   │   ├── AuthorUpdateType.java
│               │   │   └── exceptions/
│               │   │       └── AuthorNotFoundException.java
│               │   ├── infrastructure/
│               │   │   ├── persistence/
│               │   │   │   ├── AuthorRepositoryImpl.java --Avaliar necessidade dessa separação (overengineering ou não?)
│               │   │   │   ├── AuthorEntity.java --Avaliar necessidade dessa separação (overengineering ou não?)
│               │   │   │   └── AuthorMapper.java --Avaliar necessidade dessa separação (overengineering ou não?)
│               │   │   └── controllers/
│               │   │       └── AuthorController.java
│               │   │   └── messaging/
│               │   │       └── AuthorEventConsumer.java
│               │   │       └── AuthorCommandProducer.java
│               └── books/
│                   └── ...
│               └── publishers/
│                   └── ...
Strategies para lidar com cada tipo de operação
A ideia aqui é ter uma Strategy para cada operação: Criação, atualização, baixa, etc

package com.empresa.autores.application.strategies;

import com.empresa.autores.domain.model.Author;

import java.util.Map;

public interface AuthorCreationStrategy {
    void validate(Author author, BookData bookData);
    void execute(Author author, BookData bookData);
}
SubStrategies para quando cada tipo de operação tem um subtipo
A ideia aqui é ter uma sub strategy dentro da strategy, para quando uma operação tem várias operações dentro dela: Atualização que possuí inclusão de campos, alteração, ou exclusão

package com.empresa.autores.application.strategies;

import com.empresa.autores.domain.model.Author;;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
public class IndieAuthorStrategy implements AuthorCreationStrategy {
    private final BookRepository bookRepository;

    public IndieAuthorStrategy(BookRepository bookRepository) {
        this.bookRepository = bookRepository;
    }

    @Override
    public void validate(Author author, CommitBookToAuthorCommand command) {
        // Validação específica para autores indie
        if (command != null) {
            Integer pages = (Integer) command.getPages();
            if (pages != null && pages > 500) {
                throw new RuntimeException("Indie authors cannot have books with more than 500 pages");
            }
        }
    }

    @Override
    public void execute(Author author, BookData bookData) {
        if (bookData != null) {
            //Build Book and store
            bookRepository.save(book);
        }
    }
}
Factory para lidar com cada Strategy
package com.empresa.autores.application.strategies;

import com.empresa.autores.domain.model.AuthorType;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
public class AuthorStrategyFactory {
    private final Map<AuthorType, AuthorCreationStrategy> strategies;

    public AuthorStrategyFactory(IndieAuthorStrategy indie, BestSellerAuthorStrategy bestSeller) {
        this.strategies = Map.of(
            AuthorType.INDIE, indie,
            AuthorType.BEST_SELLER, bestSeller
        );
    }

    public AuthorCreationStrategy getStrategy(AuthorType type) {
        return strategies.get(type);
    }
}
Handler para gerenciar criação de autores
package com.empresa.autores.application.handlers;

import com.empresa.autores.application.commands.CreateAuthorCommand;
import com.empresa.autores.application.events.AuthorCreatedEvent;
import com.empresa.autores.application.strategies.AuthorCreationStrategy;
import com.empresa.autores.application.strategies.AuthorStrategyFactory;
import com.empresa.autores.domain.model.Author;
import com.empresa.autores.domain.model.AuthorId;
import com.empresa.autores.domain.repository.AuthorRepository;
import com.empresa.livros.application.commands.CommitBookToAuthorCommand;
import com.empresa.autores.infrastructure.messaging.AuthorCommandProducer;
import com.empresa.shared.messaging.CommandHandler;
import com.empresa.shared.messaging.EventPublisher;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

@Component
public class CreateAuthorHandler  {
    private final AuthorRepository authorRepository;
    private final AuthorStrategyFactory strategyFactory;
    private final EventPublisher eventPublisher;
    private final AuthorCommandProducer commandProducer;

    public CreateAuthorHandler(AuthorRepository authorRepository, 
                             AuthorStrategyFactory strategyFactory, 
                             EventPublisher eventPublisher,
                             AuthorCommandProducer commandProducer) {
        this.authorRepository = authorRepository;
        this.strategyFactory = strategyFactory;
        this.eventPublisher = eventPublisher;
        this.commandProducer = commandProducer;
    }

    @Override
    @Transactional
    public void handle(CreateAuthorCommand command) {
        // Cria o autor
        Author author = new Author(
            new AuthorId(command.authorId()),
            command.name(),
            command.type()
        );

        // Valida e executa estratégia específica
        AuthorCreationStrategy strategy = strategyFactory.getStrategy(command.type());
        strategy.validate(author, command.bookData());
        
        // Persiste o autor
        authorRepository.save(author);
        
        // Publica evento de autor criado
        eventPublisher.publish(new AuthorCreatedEvent(
            author.getId().getValue(),
            author.getName(),
            author.getType()
        ));
        
        // Se há dados de livro, envia comando para a fila
        if (command.bookData() != null && !command.bookData().isEmpty()) {
            CommitBookToAuthorCommand bookCommand = new CommitBookToAuthorCommand(
                author.getId().getValue(),
                author.getType(),
                command.bookData()
            );
            
            commandProducer.sendCommitBookCommand(bookCommand);
        }
    }
}
Comando para atrelar autor ao livro
package com.empresa.livros.application.commands;

import com.empresa.autores.domain.model.AuthorType;
import com.empresa.shared.messaging.Command;

import java.util.Map;
import java.util.UUID;

public record CommitBookToAuthorCommand(
    UUID authorId,
    AuthorType authorType,
    BookData bookData
) implements Command {}
Publicador de comandos para autores
package com.empresa.autores.infrastructure.messaging;

import com.empresa.livros.application.commands.CommitBookToAuthorCommand;
import com.empresa.livros.application.commands.UnlinkBooksFromAuthorCommand;
import com.empresa.shared.messaging.QueueService;
import org.springframework.stereotype.Component;

@Component
public class AuthorCommandProducer {
    private final QueueService queueService;
    
    @Value("${queue.xpto}")
    private String xpto;

    @Value("${queue.xpto}")
    private String xpto2;

    public AuthorCommandProducer(QueueService queueService) {
        this.queueService = queueService;
    }
    
    public void sendCommitBookCommand(CommitBookToAuthorCommand command) {
        queueService.sendCommand(xpto, command);
    }
    
    public void sendUnlinkBooksCommand(UnlinkBooksFromAuthorCommand command) {
        queueService.sendCommand(xpto2, command);
    }
}
Factory para consumidor de livros (a partir da fila)
package com.empresa.livros.infrastructure.messaging;

import com.empresa.livros.application.commands.CommitBookToAuthorCommand;
import com.empresa.livros.application.commands.UnlinkBooksFromAuthorCommand;
import com.empresa.livros.application.handlers.CommitBookToAuthorHandler;
import com.empresa.livros.application.handlers.UnlinkBooksFromAuthorHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
public class BookCommandConsumer {
    private final CommitBookToAuthorHandler commitHandler;
    private final UnlinkBooksFromAuthorHandler unlinkHandler;
    
    public BookCommandConsumer(CommitBookToAuthorHandler commitHandler,
                             UnlinkBooksFromAuthorHandler unlinkHandler) {
        this.commitHandler = commitHandler;
        this.unlinkHandler = unlinkHandler;
    }
    
    @SqsListener(queues = "book.commit.queue")
    public void handleCommitBookCommand(CommitBookToAuthorCommand command) {
        commitHandler.handle(command);
    }
    
    @SqsListener(queues = "book.unlink.queue")
    public void handleUnlinkBooksCommand(UnlinkBooksFromAuthorCommand command) {
        unlinkHandler.handle(command);
    }
}
Handler para lidar com comprometimento de livro
package com.empresa.livros.application.handlers;

import com.empresa.livros.application.commands.CommitBookToAuthorCommand;
import com.empresa.livros.domain.model.Book;
import com.empresa.livros.domain.model.BookId;
import com.empresa.livros.domain.repository.BookRepository;
import com.empresa.shared.messaging.CommandHandler;
import com.empresa.shared.messaging.EventPublisher;
import org.springframework.stereotype.Component;

@Component
public class CommitBookToAuthorHandler implements CommandHandler<CommitBookToAuthorCommand> {
    private final BookRepository bookRepository;
    private final EventPublisher eventPublisher;
    
    public CommitBookToAuthorHandler(BookRepository bookRepository, 
                                   EventPublisher eventPublisher) {
        this.bookRepository = bookRepository;
        this.eventPublisher = eventPublisher;
    }
    
    @Override
    public void handle(CommitBookToAuthorCommand command) {
        // Validações específicas baseadas no tipo de autor
        if (command.authorType() == AuthorType.INDIE) {
            validateIndieAuthorBook(command.bookData());
        } else if (command.authorType() == AuthorType.BEST_SELLER) {
            validateBestSellerAuthorBook(command.bookData());
        }
        
        // Cria e persiste o livro
        Book book = createBookFromData(command.authorId(), command.bookData());
        bookRepository.save(book);
        
        // Publica evento de livro criado (se necessário)
        // eventPublisher.publish(new BookCreatedEvent(...));
    }
    
    private void validateIndieAuthorBook(CommitBookToAuthorCommand bookData) {
        Integer pages = (Integer) bookData.get("pages");
        if (pages != null && pages > 500) {
            throw new RuntimeException("Indie authors cannot have books with more than 500 pages");
        }
    }
    
    private void validateBestSellerAuthorBook(CommitBookToAuthorCommand bookData) {
        String publisher = (String) bookData.get("publisher");
        if (publisher == null || publisher.isEmpty()) {
            throw new RuntimeException("Best seller authors must have a publisher");
        }
    }
    
    //Este método pode ir pra dentro de Book
    private Book createBookFromData(UUID authorId, CommitBookToAuthorCommand bookData) {
        // Lógica para criar um livro a partir dos dados
        return new Book(
            new BookId(UUID.randomUUID()),
            (String) bookData.get("title"),
            authorId,
            (Integer) bookData.get("pages"),
            (String) bookData.get("publisher")
        );
    }
}